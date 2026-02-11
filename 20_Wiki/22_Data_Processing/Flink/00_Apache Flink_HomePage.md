>[!QUOTE] 
>핵심 철학 **"Spark가 '배치'를 잘게 쪼개서 스트리밍 흉내를 낸다면, Flink는 태생부터 '스트리밍'이다."** 
>_진정한 Real-time Stream Processing과 저지연(Low Latency) 아키텍처를 향한 여정._


## 📂 Quick Access (바로가기)

|**항목**|**경로 / 링크**|**비고**|
|---|---|---|
|**Project Root**|`/Users/gong/gong_study_de/apache-flink/playground/src`|로컬 작업 공간|
|**Source Code**|[📂 Playground 폴더 열기](https://www.google.com/search?q=file:///Users/gong/gong_study_de/apache-flink/playground/src/)|실제 파이썬 코드|
|**Diagram**|[[01_Apache Flink_Flow]]|데이터 흐름도|
|**Official Docs**|[PyFlink API Reference](https://nightlies.apache.org/flink/flink-docs-stable/api/python/_modules/pyflink/datastream/functions.html)|공식 매뉴얼|

---
## Concept (이론 & 개념)

_Flink를 지탱하는 핵심 사상과 아키텍처 이해하기._

- [[Flink_Introduction]] : **Flink란 무엇인가?** (Stateful Computations over Data Streams)
- [[Batch_vs_Stream]] : **Batch vs Streaming** (Bounded vs Unbounded 데이터의 차이)
- [[Flink_vs_Spark]] : **Flink vs Spark** (마이크로 배치 vs 네이티브 스트리밍)
- [[Flink_Architecture_Overview]] : **Architecture** (JobManager와 TaskManager의 역할 분담)
- [[Flink_Execution_Models]] : **실행 모드** (Session Mode vs Application Mode)
- [[Flink_Cluster_Deployment_Modes]] : **배포 전략** (Native Kubernetes, YARN, Standalone)

---

## Environment (환경 구축)

_로컬(Mac)과 Docker를 활용한 격리된 개발 환경._

- [[Flink_Docker_Setup(PyFlink)]] : **Docker Compose Setup** (이미지 빌드, `docker-compose.yml` 설정)

---

## PyFlink Coding (코드 & 문법)

_DataStream API를 활용한 파이프라인 개발의 기초._

- [[PyFlink_Import_Analysis]] : **Import 모음+분석** (TypeInfo, Source, Environment 등 필수 모듈)
- [[PyFlink_코드 해부_common ⭐️]] : **코드 뜯어보기** (Word Count 예제로 익히는 Flink 5단계 공식)

---
## Kafka Integration (실전 연동) 

> [!check] **Why Kafka?**
> 
> 학습용 메모리 데이터만으로는 실무적인 에러 핸들링을 배우기 어려움. **"진짜 데이터 흐름"**을 만들기 위해 Kafka를 도입함.

- [[PyFlink_Kafka_Docker_Setup]] : **Kafka 환경 구축** (Docker-compose, 네트워크 격리 해결)
- [[PyFlink_Kafka_Source]] : **Source 변경** (`fromData` → `KafkaSource` 코드 마이그레이션)
- [[PyFlink + Kafka 연동 완벽 가이드 ⭐️]] : **🔥 실행 & 트러블슈팅 매뉴얼** (Job 정지/관리 명령어 포함)
- [[PyFlink_Kafka_Sink_Guide]] : **Sink 확장** (`sink_to` 데이터 내보내기)
- [[PyFlink_Kafka_코드해부_Common ⭐️]] : **Import & 문법 완전 분석** (Kafka 전용 커넥터 분석)
- [[PyFlink_Operators_Basic]] : **기본 연산자** (`map` 1:1 변환, `filter` 거름망, `flat_map` 쪼개기)
- [[PyFlink_KeyBy_DeepDive]] : **데이터 그룹핑** (분산 처리를 위한 `.key_by` 파티셔닝 원리)
- [[PyFlink_Windows]] : **데이터 슬라이싱** (무한한 스트림을 시간/개수로 자르는 `Tumbling`, `Sliding` 기법)
- [[PyFlink_Window_Functions]] : **윈도우 함수 (The Chefs)**
    - `ReduceFunction`: 입력 2개를 하나로 합치는 단순 누적 계산 (Sum, Max)
    - `AggregateFunction`: 평균(Avg)처럼 입력과 결과 타입이 다른 복합 계산 (Accumulator 활용)
    - `ProcessWindowFunction`: 모든 데이터와 윈도우 메타데이터(시작/종료 시간)에 접근
- [[PyFlink_Trigger_Watermark]] : **윈도우 제어 (Timing & Control)** 
	- **Trigger:** "데이터가 n개 모이면 발사!" (조기 출력 결정) 
	- **Watermark:** "늦게 온 데이터는 버려!" (이벤트 시간 및 지연 처리)

---
##  Operations (운영 & 모니터링)

_제출된 Job의 상태를 확인하고 관리하는 법._

- [[Flink_Dashboard_Analysis]] : **Web UI 분석** (Job Graph, Records Sent/Received 확인)