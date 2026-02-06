### 📂 [바로가기]

- **노트북 경로:** `/Users/gong/gong_study_de/apache-flink/playground/src`
- **소스코드(Src):** [Playground 폴더 열기](file:///Users/gong/gong_study_de/apache-flink/playground/src/)
- **다이어그램:** [[01_Apache Flink_Flow]]

> [!QUOTE] 핵심 철학
> **"Spark가 '배치'를 잘게 쪼개서 스트리밍 흉내를 낸다면, Flink는 태생부터 '스트리밍'이다."**
> 진정한 Event-Driven 아키텍처와 저지연(Low Latency) 처리를 위한 여정입니다.

---
##  Concept (이론 & 개념)

Flink를 다루기 위해 필수적으로 알아야 할 기초 지식입니다.

- [[Flink_Introduction]] : **Flink란 무엇인가?** (특징, 장점)
- [[Batch_vs_Stream]] : **Batch vs Streaming** (데이터 처리의 두 가지 관점)
- [[Flink_vs_Spark]] : **Flink vs Spark** (아키텍처 및 사용 사례 비교)
- [[Flink_Architecture_Overview]] : **Flink Architecture** (JobManager, TaskManager의 역할)
- [[Flink_Execution_Models]] : **실행 모드** (Session Mode vs Application Mode)

##  Environment (환경 구축)

로컬(Mac/Docker)에서 Flink를 실행하기 위한 환경 설정입니다.

- [[Flink_Docker_Setup(PyFlink)]] : **Docker Set up** (docker-compose, Dockerfile 설정, 트러블슈팅)

##  PyFlink Coding (코드 & 문법)

실제 파이썬 코드로 Flink 파이프라인을 구축하는 핵심 노트입니다.

- [[PyFlink_Import_Analysis]] : **Import 모음+분석** (Environment, Types, Source 등 필수 모듈 해설)
- [[PyFlink_코드 해부_common ⭐️]] : **코드 뜯어보기** (Word Count 예제로 배우는 5단계 공식 & `yield` 패턴)

##  Operations (운영 & 모니터링)

실행된 Job을 관리하고 결과를 확인하는 방법입니다.

- [[Flink_Dashboard_Analysis]] : **Web UI 분석** (Job Graph, Backpressure, 에러 로그 확인법)

---
