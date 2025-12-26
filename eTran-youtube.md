제공해주신 유튜브 영상은 NSDI '25에서 발표된 **eTran** 논문의 프레젠테이션입니다. 영상 내용을 분석하여 발표의 흐름과 핵심 장표(Slide) 내용을 텍스트로 재구성해 정리해 드립니다.

(※ 실제 화면 캡처 대신, 발표 자료의 핵심 텍스트와 데이터를 기반으로 슬라이드 내용을 재구성했습니다.)

### 1. 발표 개요 (Presentation Overview)

- **발표자:** Yao Wang (Xiamen University) - *NSDI '25 세션 대리 발표*
- **주제:** 데이터센터 애플리케이션의 다양화로 인해 새로운 전송 프로토콜(Transport Protocol)이 필요하지만, 리눅스 커널에 이를 반영하는 데는 너무 오랜 시간이 걸리는 문제("Ossification")를 해결하기 위한 시스템 **eTran** 제안.

### 2. 핵심 내용 분석 및 주요 장표 (Key Slides & Analysis)

#### ** 문제 제기: 커널 전송 계층의 정체 (The Problem)**

발표자는 데이터센터 워크로드가 다양해지고 있지만, 커널 전송 프로토콜의 진화 속도가 이를 따라가지 못한다고 지적합니다.

**💡 슬라이드 요약:**

- 새로운 프로토콜이 커널 메인라인에 병합되기까지 엄청난 시간이 소요됨.
- **DCTCP:** 연구(2010) → 커널 병합(2014) = **약 4년** 소요
- **MPTCP:** 연구(2009/2010) → 커널 병합(2019/2020) = **약 10년** 소요
- **Homa:** 2018년 발표되었으나, 2021년 이후에도 메인라인에 포함되지 못함.

#### ** eTran의 설계 목표 (Design Goals)**

기존의 **Kernel-Bypass(사용자 공간 전송)** 방식은 성능은 좋지만 커널의 보호(Protection) 기능을 잃는 단점이 있습니다. eTran은 이 두 마리 토끼를 잡는 것을 목표로 합니다.

| **목표**                | **설명**                                                     |
| ----------------------- | ------------------------------------------------------------ |
| **Agile Customization** | 누구나 쉽게 전송 프로토콜을 커스터마이징 가능                |
| **Kernel Safety**       | eBPF Verifier를 통해 커널 붕괴 방지                          |
| **Strong Protection**   | 전송 상태(State)를 커널 내부에 숨겨 애플리케이션으로부터 보호 |
| **High Performance**    | Kernel-Bypass에 버금가는 고성능 달성                         |

#### ** eTran 아키텍처 (Architecture Overview)**

eTran은 전송 계층을 **제어 경로(Control Path)**와 **데이터 경로(Data Path)**로 분리하고, 커널 eBPF 서브시스템을 확장했습니다.

🖥️ 아키텍처 다이어그램 내용 재구성:

[ Application ]  <-- (Segmentation/Reassembly) -->  [ User-space Lib (Untrusted) ]

| |

| (IPC via LRPC) | (AF_XDP)

v                                                      v

(Root Privilege)                                        (Safe Runtime)

- Flow Management                                      - XDP Hook (Ingress)
- Complex Congestion Control                           - XDP_EGRESS Hook (Egress/Pacing)
- Loss Recovery (Timeout)                              - XDP_GEN Hook (Packet Gen)
- Transport States (Maps)                              - Rate/Window Enforcer

- **핵심 혁신:** 기존 eBPF에는 없던 `XDP_EGRESS`(송신 제어)와 `XDP_GEN`(패킷 생성) 훅을 추가하여 전송 프로토콜 구현을 가능하게 함.1

  

  

#### ** 성능 평가: Homa 프로토콜 (Evaluation: Homa)**

리눅스 커널 모듈로 구현된 Homa와 eTran으로 구현된 Homa의 성능을 RPC 워크로드에서 비교했습니다.

**📊 그래프 데이터 요약:**

- **P99 Tail Latency:** eTran이 Linux Homa 대비 **7.5배 더 낮은 지연 시간** 달성 (짧은 메시지 위주 워크로드).

- **P50 Latency:** Linux Homa 대비 **3.6배 더 빠름**.

- **원인:** `sk_buff` 할당 제거 및 시스템 콜 오버헤드 최소화 덕분.2

  

  

#### ** 성능 평가: TCP (Evaluation: TCP & TAS)**

Key-Value Store 애플리케이션을 사용하여 Linux TCP, eTran TCP, 그리고 고성능 커널 바이패스 시스템인 TAS를 비교했습니다.

**📊 그래프 데이터 요약:**

- **Throughput:** Linux TCP 대비 **2.4배 ~ 4.8배 높은 처리량**.

- **Latency:** Linux TCP 대비 **P50은 3.7배**, **P99는 3.2배** 더 낮음.

- **vs TAS:** 커널 바이패스 방식인 TAS보다는 약간 느리지만(Slightly worse), 커널의 안정성을 유지하면서 대등한 수준의 지연 시간을 달성함.1

  

  

### 3. 질의응답 (Q&A Highlights)

발표 후 이어진 Q&A 세션에서 논의된 주요 내용입니다.

1. **Q: eTran으로 새 프로토콜을 구현하는 것이 쉬운가? (코드량이 7.5k 라인이나 되는데?)**

   - **A:** 구현 자체가 쉬워지는 것이 목표는 아님. 목표는 로직을 커널로 오프로드하여 보안과 성능을 얻는 것임. 하지만 DCTCP나 Homa 같은 기본 템플릿을 제공하므로, 이를 수정하여 변형(Variant)을 만드는 것은 훨씬 쉬움.3

     

     

2. **Q: 구현 불가능한 프로토콜이 있는가?**

   - **A:** 대부분의 전송 프로토콜(세그멘테이션, 페이싱, 혼잡 제어 등)이 공통적인 추상화를 공유하므로, HPCC 등 다양한 프로토콜 구현이 가능함.3

     

     

3. **Q: Meta의 MV Fast 같은 사용자 공간 라이브러리와의 차이점은?**

   - **A:** 핵심 차이는 **보안(Security)**임. 사용자 공간 라이브러리는 애플리케이션이 전송 상태를 조작할 수 있지만, eTran은 커널 내부에서 상태를 보호함. 또한 폴링(Polling) 대신 인터럽트 방식을 사용하여 CPU 효율성이 높음.3

     

     

4. **Q: 커널 커뮤니티의 반응은? (Upstreaming 가능성)**

   - **A:** 커널 커뮤니티와 소통 중이나, eBPF 훅을 추가하는 것은 매우 도전적인 과정임.3

     

     

이 분석 내용이 현재 진행 중인 과제 발굴 및 eTran 분석에 도움이 되기를 바랍니다. 추가적인 세부 사항이나 특정 데이터가 필요하시면 말씀해 주세요.
