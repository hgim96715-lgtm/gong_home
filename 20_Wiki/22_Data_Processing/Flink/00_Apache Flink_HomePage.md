> [!QUOTE]
> 핵심 철학 **"Spark가 '배치'를 잘게 쪼개서 스트리밍 흉내를 낸다면, Flink는 태생부터 '스트리밍'이다."**
> _진정한 Real-time Stream Processing과 저지연(Low Latency) 아키텍처를 향한 여정._

## 📂 Quick Access (바로가기)

| **항목** | **경로 / 링크** | **비고** |
|---------|--------------|---------|
| **Project Root** | `/Users/gong/gong_study_de/apache-flink/playground/src` | 로컬 작업 공간 |
| **Source Code** | [📂 Playground 폴더 열기](https://www.google.com/search?q=file:///Users/gong/gong_study_de/apache-flink/playground/src/) | 실제 파이썬 코드 |
| **Diagram** | [[01_Apache Flink_Flow]] | 데이터 흐름도 |
| **Official Docs** | [PyFlink API Reference](https://nightlies.apache.org/flink/flink-docs-stable/api/python/_modules/pyflink/datastream/functions.html) | 공식 매뉴얼 |

---

## Concept (이론 & 개념)

_Flink를 지탱하는 핵심 사상과 아키텍처 이해하기._

- [[Flink_Introduction]] : Flink란 무엇인가? (`Stateful Computations`, `Data Stream`, `저지연`, `이벤트 드리븐`, `Exactly-once`)
- [[Batch_vs_Stream]] : Batch vs Streaming (`Bounded`, `Unbounded`, `마이크로 배치`, `네이티브 스트리밍`, `처리 지연`)
- [[Flink_vs_Spark]] : Flink vs Spark (`마이크로 배치 vs 네이티브 스트리밍`, `지연시간 비교`, `상태 관리`, `적합한 use case`)
- [[Flink_Architecture_Overview]] : Architecture (`JobManager`, `TaskManager`, `Task Slot`, `Dispatcher`, `ResourceManager`)
- [[Flink_Execution_Models]] : 실행 모드 (`Session Mode`, `Application Mode`, `Per-Job Mode`, `클러스터 생명주기`)
- [[Flink_Cluster_Deployment_Modes]] : 배포 전략 (`Kubernetes`, `YARN`, `Standalone`, `Docker`, `클라우드 배포`)

---

## Environment (환경 구축)

_로컬(Mac)과 Docker를 활용한 격리된 개발 환경._

- [[Flink_Docker_Setup(PyFlink)]] : Docker Compose Setup (`docker-compose.yml`, `이미지 빌드`, `JobManager`, `TaskManager`, `포트 8081`)
- [[PyFlink_Kafka_Docker_Setup]] : Kafka 환경 구축 (`docker-compose`, `네트워크 격리`, `KAFKA_ADVERTISED_LISTENERS`, `브릿지 네트워크`, `트러블슈팅`)

---

## PyFlink Basic (기초 문법)

_DataStream API를 활용한 파이프라인 개발의 기초._

- [[PyFlink_Import_Analysis]] : Import 모음+분석 (`StreamExecutionEnvironment`, `TypeInfo`, `Source`, `Sink`, `필수 모듈`, `의존성`)
- [[PyFlink_코드해부_common ⭐️]] : 코드 뜯어보기 (`Word Count`, `환경 설정`, `Source 정의`, `변환`, `Sink`, `execute`, `Flink 5단계 공식`)
- [[PyFlink_Operators_Basic]] : 기본 연산자 (`map 1:1변환`, `filter 거름망`, `flat_map 쪼개기`, `DataStream`, `람다 함수`)

---

## Kafka Integration (실전 데이터 파이프라인)

> [!check] **Why Kafka?**
> 학습용 메모리 데이터만으로는 실무적인 에러 핸들링을 배우기 어려움. **"진짜 데이터 흐름"**을 만들기 위해 Kafka를 도입함.
> JAVA → PyFlink의 한계로 Kafka + PyFlink 활용

- [[PyFlink_Kafka_Source]] : Source 변경 (`fromData → KafkaSource`, `마이그레이션`, `bootstrap.servers`, `group.id`, `auto.offset.reset`)
- [[PyFlink_Kafka_Sink_Guide]] : Sink 확장 (`KafkaSink`, `sink_to`, `SerializationSchema`, `delivery_guarantee`, `데이터 내보내기`)
- [[PyFlink + Kafka 연동 완벽 가이드 ⭐️]] : 🔥 실행 & 트러블슈팅 매뉴얼 (`Job 제출`, `Job 정지`, `flink run`, `취소 명령어`, `에러 해결`)
- [[PyFlink_Kafka_코드해부_Common ⭐️]] : Import & 문법 완전 분석 (`Kafka 전용 커넥터`, `jar 의존성`, `FlinkKafkaConsumer`, `오프셋 관리`)

---

## Window & Time (시간 제어)

_무한한 스트림 데이터를 시간/개수 단위로 잘라서 처리하는 핵심 기술._

- [[PyFlink_KeyBy_DeepDive]] : 데이터 그룹핑 (`key_by`, `파티셔닝`, `해시 기반 분산`, `KeyedStream`, `분산 처리 원리`)
- [[PyFlink_Windows]] : 데이터 슬라이싱 (`Tumbling Window`, `Sliding Window`, `Session Window`, `시간 단위 잘라내기`)
- [[PyFlink_Window_Functions]] : 윈도우 함수 The Chefs
    - `Reduce` : 입력 2개를 합치는 누적 계산 (`sum`, `max`, `동일 타입 입출력`, `중간 집계`)
    - `Aggregate` : 입력과 결과 타입이 다른 복합 계산 (`AggregateFunction`, `accumulator`, `avg`, `타입 변환`)
    - `Process` : 메타데이터(시간)까지 다루는 만능 함수 (`ProcessWindowFunction`, `Context`, `윈도우 시간 접근`, `Side Output`)
- [[PyFlink_Trigger_Watermark]] : 타이밍 제어
    - **Trigger** : "데이터 n개 모이면 발사!" (`CountTrigger`, `조기 출력`, `EventTimeTrigger`, `발사 조건`)
    - **Watermark** : "이벤트 시간 기준 지연 처리" (`WatermarkStrategy`, `이벤트 타임`, `Late Data`, `지연 허용`)
- [[PyFlink_Allowed_Lateness]] : 지각생 구제 (`allowed_lateness`, `닫힌 윈도우 재오픈`, `Side Output`, `OutputTag`, `늦은 데이터 처리`)
- [[PyFlink_KeyedProcessFunction]] : 키 기반 이벤트 처리 (`KeyedProcessFunction`, `onTimer`, `타이머 등록`, `상태 접근`, `이벤트 드리븐`)

---

## State & Fault Tolerance (고급 아키텍처)

_Flink가 "Stateful"한 이유. 장애가 발생해도 데이터를 잃지 않는 비결._

- [[Flink_State]] : 상태 관리 Memory (`Keyed State`, `Operator State`, `ValueState`, `ListState`, `MapState`, `이벤트 간 기억`)
- [[Flink_State_Persistence]] : 영구 저장과 복구 Safety
    - **Checkpoint** : Flink가 알아서 찍는 자동 스냅샷 (`장애 복구`, `checkpointInterval`, `Exactly-once`, `자동화`)
    - **Savepoint** : 사람이 직접 찍는 수동 스냅샷 (`배포`, `업그레이드`, `수동 트리거`, `버전 관리`)
    - **State Backend** : 상태가 저장되는 위치 (`HashMapStateBackend`, `RocksDB`, `Memory vs Disk`, `대용량 상태`)
- [[PyFlink_State_and_RuntimeContext]] : State 선언 공식 (`RuntimeContext`, `ValueStateDescriptor`, `get_state`, `get_runtime_context`, `3단계 문법`)

---

## Operations (운영 & 모니터링)

_제출된 Job의 상태를 확인하고 관리하는 법._

- [[Flink_Dashboard_Analysis]] : Web UI 분석 (`Job Graph`, `Records Sent/Received`, `Back Pressure`, `Checkpoint 현황`, `localhost:8081`)

---

## Flink SQL & Table API (고수준 추상화)

> 복잡한 코딩 없이 SQL 쿼리와 유사한 방식으로 스트림을 처리하는 고수준 API.

- [[PyFlink_Table_API_Intro]] : Table API란? (`TableEnvironment`, `Table`, `DataStream 변환`, `고수준 추상화`, `배치/스트림 통합`)
- [[PyFlink_Table_Expressions]] : 기본 연산과 표현식 (`select`, `filter`, `col`, `lit`, `조건 표현식`, `컬럼 연산`)
- [[PyFlink_SQL_Integration]] : SQL 쿼리 사용하기 (`execute_sql`, `create_temporary_view`, `SQL DDL`, `Table API 혼용`)
- [[PyFlink_SQL_Windows]] : Window in SQL (`TUMBLE`, `HOP`, `SESSION`, `GROUP BY`, `윈도우 집계`, `시간 컬럼`)
- **[[PyFlink_SQL_Watermark]]** : Table SQL Watermark (`WATERMARK FOR`, `rowtime`, `지연 허용`, `이벤트 타임 정의`, `DDL 선언`)