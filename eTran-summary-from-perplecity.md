이 논문은 eBPF를 활용해 커널 안에서 다양한 트랜스포트 프로토콜을 ‘확장 가능하게’ 구현할 수 있는 새로운 시스템 eTran을 제안하고, TCP(DCTCP)와 Homa로 그 효과를 검증하는 작업이다. 기존 커널 TCP/Homa 대비 대역폭·지연에서 큰 개선을 보이면서도, 커널 바이패스(예: TAS) 수준에 근접한 성능을 달성하는 것이 핵심 포인트다.[1]

## 문제의식과 목표

- 데이터센터 워크로드의 다양화(마이크로서비스 RPC, 스토리지, 분산 학습 등)로 새로운 트랜스포트 설계가 많이 나오지만, 리눅스 커널 스택에 반영되는 데 수년이 걸리거나 아예 병합되지 않는 문제가 있다(DCTCP 4년, MPTCP 10년, Homa 미병합).  
- 이 때문에 많은 시스템이 유저 공간/하드웨어 트랜스포트를 사용하지만, 이는 보호·관리 측면에서 커널 스택의 장점을 잃게 된다.  
- 논문 목표는 커널 트랜스포트를 **확장 가능**하게 만들어,  
  - 새로운 트랜스포트 설계를 민첩하게 구현·배포하고,  
  - 커널 수준의 안전성과 보호를 유지하면서,  
  - 커널 바이패스에 근접하는 고성능을 제공하는 것이다.  

## eTran의 핵심 아이디어와 아키텍처

### 전체 구조

- eTran은 트랜스포트를 **Control path**와 **Data path**로 나누고, eBPF + AF_XDP 조합으로 커널 안에 트랜스포트 상태와 핵심 로직을 넣되, 데이터 IO는 유저 공간으로 직접 넘긴다.  
- Control path (root daemon):  
  - eBPF 프로그램 로딩/부착, AF_XDP 소켓·UMEM 생성, 연결 관리(TCP handshake 등), 복잡한 CC(부동소수점), 타임아웃 기반 재전송 등 비성능 크리티컬 기능을 처리.  
- Data path:  
  - 커널(eBPF) 측: 패킷 재조립, ACK/credit 처리, pacing, 빠른 재전송 등 성능 크리티컬 로직과 **모든 트랜스포트 상태**를 커널 내부 eBPF map에 유지.  
  - 유저 공간: AF_XDP를 통해 패킷을 받고, 이를 flow/RPC 단위로 조립·분해하는 얇은 라이브러리. 이 라이브러리는 커널의 트랜스포트 상태에는 직접 접근하지 못한다.  

### eBPF 확장: 두 개의 새 hook과 하나의 새 map

eTran이 기존 eBPF/XDP만으로는 구현하기 어려운 세 가지 문제(송신 시 상태 업데이트, 커널 내 패킷 생성, pacing용 버퍼링)를 해결하기 위해 커널에 다음을 추가한다.

- XDP_EGRESS hook:  
  - AF_XDP TX ring에서 NIC 드라이버로 패킷이 넘어가기 직전에 실행되는 eBPF hook.  
  - 기존 XDP와 동일한 컨텍스트(struct xdp_md)를 사용하며, XDP_TX/XDP_REDIRECT/XDP_DROP 액션을 지원.  
  - 송신 패킷에 대해 헤더 구성, 상태 업데이트, window/credit 검사, pacing 큐로의 enqueue 등을 수행.  

- XDP_GEN hook:  
  - NAPI poll 종료 시점(xdp_do_flush)에서 호출되며, 미리 쌓아 둔 메타데이터를 기반으로 ACK/credit 패킷을 커널 내에서 생성한다.  
  - page_pool 기반으로 frame을 미리 할당하고 budget 내에서 루프를 돌며 batched ACK/credit을 생성·송신하여 오버헤드를 줄인다.  

- BPF_MAP_TYPE_PKT_QUEUE (PKT_QUEUE):  
  - pacing을 위한 큐 자료구조로, 여러 버킷과 각 버킷 안에 xdp_frame 포인터를 저장하는 슬롯들을 가진다.  
  - dynptr를 이용해 eBPF에서 안전하게 패킷 내용을 접근·수정할 수 있으며, 다양한 pacing 정책(속도 기반, credit 기반)을 구현하는 backbone 역할을 한다.  

### pacing과 비동기 전송

- BPF timer의 callback 메커니즘을 재활용하여, 특정 버킷/큐에 쌓인 패킷을 softirq 혹은 전용 kthread에서 비동기로 꺼내 전송한다.  
- Rate-based pacing: Carousel과 유사한 timing wheel 방식으로 tx_timestamp를 버킷에 매핑.  
- Credit-based pacing: Homa처럼 RPC별 버킷에 패킷을 쌓아 두고 credit이 replenishment될 때 해당 버킷을 비운다.  
- Out-of-order completion을 지원하기 위해 AF_XDP의 버퍼 재사용 로직을 수정, pacing으로 인해 송신 순서가 바뀌더라도 올바르게 completion을 처리할 수 있게 한다.  

### 유저 공간 IO 설계

- AF_XDP 제약(소켓당 1 NIC queue)을 우회하기 위해 **Virtual AF_XDP socket**을 도입해 여러 실제 AF_XDP 소켓을 epoll로 묶어 하나의 논리 소켓처럼 사용한다.  
- 상단에는 두 가지 API를 제공:  
  - POSIX socket API (read/write, epoll) → LD_PRELOAD로 기존 앱을 수정 없이 eTran으로 전환 가능.  
  - RPC-style API (continuation 콜백 기반), eRPC 스타일의 event-driven 모델.  
- 여러 NIC queue와 스레드 사이의 공정한 패킷 처리 위해 DRR 스케줄링을 사용하고, Fill/Comp ring은 per-queue spinlock으로 동기화하여 동시 접근을 제어한다.  

### 멀티 테넌시와 트래픽 관리

- 애플리케이션마다 eBPF map namespace 및 NIC queue를 분리하여 IO 버퍼 접근을 격리하고, 문제 있는 앱의 큐를 다른 CPU로 스케줄링하는 등 소프트웨어 기반 멀티 테넌시를 지원한다.  
- 모든 패킷이 eBPF hook을 통과하므로, 트래픽 모니터링/ACL/rate limiting 같은 정책을 모듈화된 eBPF 체인(bpf_tail_call)으로 붙여 유연하게 관리할 수 있다.  

## TCP (DCTCP)와 Homa 구현

### TCP (DCTCP) on eTran

- Control path:  
  - 자체 AF_XDP 소켓으로 SYN/SYN-ACK 핸들링, 3-way handshake 수행.  
  - 커널 eBPF map에 connection state를 저장하고, CC 상태는 별도 eBPF array로 분리하여 BPF_F_MMAPABLE로 데몬이 직접 mmap 후 업데이트(시스템콜 오버헤드 감소).  
  - DCTCP 등 복잡한 CC는 CCP 아키텍처처럼 data path 밖에서 실행(부동소수점 사용, 복잡한 연산).  

- Data path (XDP, XDP_EGRESS):  
  - XDP: TCP 데이터 패킷에 대해 4-tuple로 연결 상태를 조회하여 seq/ack 검증, 윈도우 업데이트, in-order는 바로 유저 공간으로, out-of-order는 TAS와 유사한 out-of-order receive 버퍼에 보관.  
  - XDP_EGRESS:  
    - connection state를 참조해 TCP/IP/Ethernet 헤더를 구성, FIB lookup 결과를 캐시해 성능 최적화.  
    - flow/CC 윈도우가 허용하면 즉시 송신(XDP_TX), 아니면 PKT_QUEUE에 enqueue(XDP_REDIRECT) 후 pacing에 의해 비동기 송신.  
    - umem_id를 체크해 다른 애플리케이션의 상태를 임의로 건드리는 잘못된 패킷을 차단한다.  

- 손실 복구:  
  - fast retransmit: XDP에서 3 duplicated ACK 감지 시 eBPF가 상태 롤백 메타데이터를 ACK에 piggyback하여 유저 공간 라이브러리로 전달, 거기서 재전송 수행.  
  - timeout 기반 재전송: Control daemon이 timeout을 감지하면 dummy 패킷을 XDP_EGRESS로 흘려보내 fast retransmit 로직을 트리거하고 곧바로 드롭, lock 없이 경로 간 동기화.  

### Homa on eTran

- Homa는 receiver-driven credit scheduling + SRPT 기반 RPC 트랜스포트로, eTran에서는 상당 부분을 eBPF data path에 올린다.  

- Control path:  
  - 소켓 생성/바인딩/종료 등 관리.  
  - 타이머 스레드가 eBPF hashmap의 RPC 상태를 batch lookup(bpf_map_lookup_batch)으로 스캔하여 timeout을 감지하고, RESEND 패킷을 보내 유저 공간에서 재전송을 트리거.  

- Data path:  
  - XDP에서 첫 unscheduled 패킷 도착 시 RPC 상태를 생성하고, RPC ID·remaining bytes 등을 갖는 CC state를 bpf_rbtree 기반 credit list에 넣는다. 이후 패킷 도착 시 remaining bytes 업데이트.  
  - XDP_GEN에서 수행하는 receiver-side credit scheduling:  
    - bpf_rbtree_lower_bound라는 새 kfunc를 추가해, 우선순위에 따라 RPC를 선택하되 peer별 하나만 선택하는 Homa 요구사항을 만족.  
    - eBPF instruction limit을 넘지 않도록 복잡한 로직을 여러 프로그램으로 쪼개고, bpf_tail_call과 per-CPU 변수(bss 섹션)를 이용해 상태를 전달.  
  - XDP_EGRESS + pacing callback에서 sender-side SRPT를 구현하기 위해, remaining bytes 기준으로 정렬된 또 다른 rbtree 위에 queued RPC들을 스케줄링.  

## 구현 규모와 안전성

- 전체 코드베이스는 약 24K LoC이며, 커널 쪽은 두 새 hook, PKT_QUEUE 및 kfunc, rbtree용 새 API(bpf_rbtree_lower_bound), AF_XDP out-of-order completion 지원이 포함된다.  
- mlx5 드라이버 변경은 약 20 LoC로 최소화하여, pacing 큐에 들어간 frame 주소를 추적하고 실제 전송 시점에 completion ring에 올바르게 반영하도록 한다.  
- eBPF verifier는 수정하지 않고, 새 hook과 kfunc를 기존 정책에 맞게 등록한다.  
  - 모든 kfunc는 softirq 컨텍스트에서 안전하게 실행될 수 있도록 비선점(non-preemptible)·유한 실행시간을 보장해야 한다.  
  - 노출되는 helper/kfunc는 최소화하여, 예를 들어 XDP_GEN에는 bpf_redirect_map을 제공하지 않는 등 보안 위험을 줄인다.  

## 성능 평가 요약

### Homa: 커널 모듈 대비

- 테스트 환경: CloudLab 10노드, 25 Gbps NIC, Linux 6.6.0, MTU 1500, TSO는 Linux Homa/TCP에만 활성화.  
- 마이크로벤치마크 결과:  
  - 32B RPC median RTT: eTran(Homa) 11.8µs vs Linux(Homa) 15.6µs.  
  - 1MB 메시지 단일 스트림 throughput: 17.7 Gbps vs 14.5 Gbps.  
  - 다중 클라이언트/서버 환경에서 RPC rate: 클라이언트 기준 2.9M ops vs 1.7M, 서버 3.3M vs 1.8M (약 1.7–1.8배).  

- 클러스터 워크로드 W2–W5:  
  - 짧은 메시지 위주인 W2/W3에서 P50/P99 RTT slowdown이 Linux(Homa)에 비해 1.4–3.6×(P50), 3.9–7.5×(P99) 개선.  
  - 큰 메시지 위주 W4/W5에서도 전반적으로 더 낮은 RTT를 보이며, 특히 가장 짧은 10% 메시지에 대해 P50/P99가 각각 4.1×/4.3× (W4), 3.9×/2.9× (W5) 개선.  
  - 일부 큰 메시지 tail에서 eTran(Homa)의 스케줄링이 미조정되어 tail latency가 튀는 경우가 있어, 향후 eBPF 기반 스케줄링 정책으로 개선 계획을 언급.  

### TCP (DCTCP): Linux TCP 및 TAS 대비

- 동일 바이너리 애플리케이션(에코 서버, KV 스토어)을 서로 다른 POSIX 구현(Linux, TAS, eTran)에 링크해 공정 비교.  

- 메시지 throughput:  
  - 큰 메시지 단일 스트림에서 eTran(TCP)은 Linux TCP와 비슷하거나 약간 우수하며, TAS보다는 다소 뒤처진다.  
  - 64 outstanding small messages(100 connection, 5 client × 5 thread)에서 1KB 메시지 기준: eTran(TCP)는 Linux TCP 대비 4.8×, TAS는 7.7× 높은 throughput.  
  - 2KB 메시지에서는 eTran(TCP)가 TAS throughput의 약 87% 수준까지 근접.  

- 연결 확장성:  
  - 64B 요청을 persistent connection으로 여러 개 유지할 때, 1K connection에서 eTran(TCP)는 Linux TCP의 2.26×, TAS는 4.1× throughput을 달성.  
  - Linux는 syscall 오버헤드와 TSO/GRO의 한계로 connection 수가 많을수록 급격히 성능이 나빠지고, eTran은 batched 처리로 이를 완화하지만, AF_XDP vs DPDK의 성능 차이 때문에 TAS보다는 낮다.  

## 요약 관점: 실무 적용 시 의미

- 이 논문이 제안하는 eTran은 “커널 안에서 eBPF로 트랜스포트를 구현·커스터마이징하는 프레임워크”로 볼 수 있다.  
- 핵심 가치:  
  - 새로운 트랜스포트(TCP 변종, Homa류 RPC 프로토콜 등)를 커널 모듈 수준의 위험 없이 빠르게 배포 가능.  
  - 커널 스택의 보호/관리(멀티 테넌시, 방화벽, rate limit)를 유지하면서도, 유저 공간 트랜스포트에 근접한 성능을 얻는다.  
  - eBPF hook/map 확장을 통해, 향후 다른 프로토콜·워크로드(예: ML 전용 트랜스포트)도 같은 프레임워크 위에서 구현 가능성을 제시한다.  

원하면, 이후에는 이 구조를 기준으로 “실제 리눅스 환경에서 eTran 스타일의 디자인을 모사하려면 어떤 커스텀 eBPF/AF_XDP 인프라를 깔아야 할지” 같은 실무 관점으로 재해석해 줄 수도 있다.

[1](https://www.usenix.org/system/files/nsdi25-chen-zhongjie.pdf)
