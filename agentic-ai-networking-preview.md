대분류	항목	상세 내용
1. 과제 개요	과제명	AI-Driven Overlay Path Optimization (AI 기반 지능형 오버레이 경로 최적화)
	핵심 컨셉	물리 망(Underlay) 변경 없이, **가상화 계층(Overlay)**에서 AI 에이전트가 경로를 자율 제어
	해결 문제	• ECMP 해시 충돌: 대용량(Elephant) 플로우가 동일 경로로 쏠려 발생하는 패킷 드랍• 운영 경직성: 물리 장비 설정을 실시간으로 변경하기 어려운 리스크 극복
2. 아키텍처(Agent Loop)	① 인지 (Sense)	• INT (In-band Network Telemetry): 패킷에 타임스탬프를 실어 실시간 지연/큐 상태 측정• Micro-burst Detection: 밀리초(ms) 단위의 순간적인 트래픽 폭주 감지
	② 판단 (Reason)	• Multi-Agent RL (MARL): 스위치별 독립 에이전트가 협력하여 최적 경로 학습• Hierarchical RL: 상위(트래픽 분류) - 하위(경로 지정) 계층 구조로 판단 속도 향상
	③ 행동 (Act)	• Traffic Steering: VXLAN/UDP 헤더 조작을 통한 경로 변경• Flowlet Switching: 대용량 플로우를 잘게 쪼개어(Flowlet) 다중 경로로 분산 전송
3. 핵심 기술	데이터 플레인	P4 / Programmable ASIC (Intel Tofino, Broadcom Trident 4 등)
	AI 모델	경량화된 강화학습(RL) 모델 (On-Switch Inference 추구)
	프로토콜	VXLAN, GENEVE (오버레이 터널링 프로토콜 활용)
4. 목표 성과	정량적 목표	• FCT (완료 시간): 30% 단축• Tail Latency (P99): 50% 개선• Packet Drop: Zero화 (재전송 최소화)
5. 벤치마킹	관련 연구	• AuTO (SIGCOMM): 계층적 RL 아키텍처 참조• AWS SRD: 다중 경로(Multi-path) 전송 방식 차용• Google Jupiter: 중앙 집중형 트래픽 엔지니어링 개념 비교
6. 로드맵	단계별 계획	1년차: 시뮬레이션(NS-3/Mininet) 및 알고리즘 설계2년차: 가상 환경(Digital Twin) 검증 및 모델 고도화3년차: 실제 P4 스위치 환경 PoC 및 상용화 검토
