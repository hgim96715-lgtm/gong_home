### 📂 [바로가기]

- **노트북 경로:** `/Users/gong/gong_study_de/apache-spark/notebooks`
- 👉 [폴더 바로 열기](file:///Users/gong/gong_study_de/apache-spark/notebooks)
- **다이어그램:** **[[01_Spark_Master_Map]]**

---

## 아키텍처 및 핵심 원리 (Architecture & Internals)

> 스파크가 **"어떻게, 그리고 왜"** 이렇게 동작하는지 설명합니다.

- **기초 개념**
    - **[[Spark_Concept_Evolution]]** : 하둡(MapReduce)과 스파크의 차이점 (`MapReduce`, `In-Memory`, `Disk vs Memory`, `DAG 실행 엔진`)
    - **[[Spark_Architecture]]** : Driver, Executor, Cluster Manager의 역할 완벽 정리 (`Driver`, `Executor`, `Cluster Manager`, `YARN`, `Standalone`, `자원 배분`)
    - **[[RDD_Concept]]** : 스파크 데이터의 불변성과 복구 능력 (`RDD`, `불변성`, `Lineage`, `복구`, `파티션`, `분산 데이터셋`)
    - **[[Spark_Job_Scheduling]]** : 자원 배분 전쟁 (`Static`, `Dynamic`, `FIFO`, `FAIR`, `Job`, `Stage`, `Task`)

- **실행 엔진 (Engine)**
    - **[[Transformations_vs_Actions]]** : Lazy Evaluation과 의존성 (`Lazy Evaluation`, `Narrow`, `Wide`, `셔플`, `Lineage`, `Action 트리거`)
    - **[[Spark_Catalyst_Optimizer]]** : 스파크의 뇌, 논리적 계획이 물리적 계획으로 바뀌는 과정 (`Catalyst`, `논리 계획`, `물리 계획`, `최적화`, `Predicate Pushdown`)
    - **[[Spark_Memory_Management]]** : Executor 메모리 구조 (`Unified Memory`, `Storage Memory`, `Execution Memory`, `OOM`, `메모리 튜닝`)

---

## 데이터 엔지니어링 (Data Engineering)

> 데이터를 **읽고, 쓰고, 변환하는** 실무 기술들을 모았습니다.

- **설정 및 시작**
    - **[[Spark_Installation_Local]]** : 로컬/Docker 환경 구축 (`pip install pyspark`, `Docker Compose`, `JAVA_HOME`, `spark-defaults.conf`)
    - **[[PySpark_Session_Context]]** : `SparkSession` 객체 생성과 필수 설정 (`SparkSession`, `builder`, `appName`, `master`, `getOrCreate`)
    - **[[Spark_Session_Deep_Dive]]** : `spark-submit`, Master, Deploy Mode 심층 분석 (`spark-submit`, `--master`, `client mode`, `cluster mode`, `--deploy-mode`)

- **DataFrame & SQL (메인 무기)**
    - **[[Spark_DataFrame_Basics]]** : 기본 조작 (`read`, `show`, `schema`, `printSchema`, `count`, `dtypes`)
    - **[[Spark_Data_IO]]** : 저장의 기술 (`Parquet`, `CSV`, `partitionBy`, `write`, `mode`, `overwrite`, `append`)
    - **[[DataFrame_Transform_Basic]]** : 조회와 필터링 (`select`, `filter`, `where`, `withColumn`, `cast`, `Column`, `alias`)
    - **[[DataFrame_Aggregation]]** : 집계와 순위 (`groupBy`, `agg`, `sort`, `orderBy`, `count`, `sum`, `avg`, `max`)
    - **[[Spark_DataFrame_Joins]]** : 조인 전략 총정리 (`inner`, `left`, `semi`, `anti`, `broadcast`, `셔플 조인`)
    - **[[Spark_Data_Cleaning]]** : 결측치와 날짜 처리 (`na.drop`, `na.fill`, `format_number`, `to_date`, `date_format`)
    - **[[SQL_with_Spark]]** : Python 대신 SQL 문법 사용하기 (`createOrReplaceTempView`, `spark.sql`, `TempView`, `SQL 혼용`)
    - **[[Spark_Functions_Library]]** : 자주 쓰는 내장 함수 사전 (`F.col`, `F.lit`, `F.when`, `F.coalesce`, `F.regexp_extract`, `Reference`)
    - **[[Spark_JSON_Handling]]** : JSON 데이터 파싱과 생성 (`from_json`, `to_json`, `schema 정의`, `StructType`, `중첩 JSON`)

- **RDD 로우 레벨 제어 (Low-Level)**
    - **[[Spark_General_Transformations]]** : 기본 변환 (`map`, `flatMap`, `filter`, `distinct`, `union`, `Lazy`)
    - **[[Spark_Key_Transformations]]** : 키(Key) 기준 제어 (`reduceByKey`, `groupByKey`, `sortByKey`, `PairRDD`, `셔플 발생`)
    - **[[Spark_Value_Transformations]]** : 값(Value)만 안전하게 변경 (`mapValues`, `flatMapValues`, `셔플 없음`, `Key 유지`)

- **외부 데이터 연동 (External Data Sources)**
    - **[[Spark_JDBC_Postgres_Guide]]** : PostgreSQL 연동 완벽 가이드 (`JDBC`, `드라이버 jar`, `url`, `dbtable`, `user`, `password`, `Docker 설정`)

---

## 성능 최적화 (Optimization & Tuning)

> **"왜 느리지?"** 싶을 때 열어보는 튜닝 비법서입니다.

- **파티셔닝 (Partitioning)**
    - **[[Spark_Partitioning_Concept]]** : 파티셔닝 기초 (`repartition`, `coalesce`, `파티션 수`, `셔플 발생 여부`, `병렬성`)
    - **[[Spark_AQE_Deep_Dive]]** : 실행 중에 계획을 바꾸는 AQE (`Adaptive Query Execution`, `Skew Join`, `자동 파티션 축소`, `런타임 최적화`)
    - **[[Spark_Dynamic_Partition_Pruning]]** : 조인할 때 필요한 데이터만 읽는 기술 (`DPP`, `파티션 프루닝`, `조인 최적화`, `불필요한 스캔 제거`)

- **전략 및 테크닉**
    - **[[Spark_Caching_Strategies]]** : 반복 계산을 막는 캐싱 (`cache`, `persist`, `StorageLevel`, `unpersist`, `MEMORY_AND_DISK`)
    - **[[Spark_Core_Broadcast]]** : 셔플을 없애는 기술 (`Broadcast Variable`, `broadcast join`, `sc.broadcast`, `셔플 제거`, `작은 테이블`)
    - **[[Spark_SQL_Hints]]** : 옵티마이저의 판단을 강제로 덮어쓰기 (`BROADCAST`, `MERGE`, `SHUFFLE_HASH`, `hint`, `강제 실행 계획`)
    - **[[Spark_Speculative_Execution]]** : 느린 태스크(낙오자) 버리기 전략 (`spark.speculation`, `낙오자 태스크`, `중복 실행`, `straggler`)
    - **[[Spark_Accumulator]]** : 분산 환경의 전역 카운터 (`sc.accumulator`, `전역 집계`, `디버깅`, `사이드 이펙트`, `add`)
    - **[[Spark_Iterating_Data]]** : 반복문의 정석 (`collect 금지`, `foreachPartition`, `mapPartitions`, `드라이버 OOM`, `분산 처리 유지`)

---

## 트러블슈팅 및 모니터링 (Troubleshooting)

> 에러 로그를 해석하고 해결하는 방법입니다.

- **분석 도구**
    - **[[Spark_UI_Guide]]** : 스파크의 MRI, Event Timeline 색깔별 의미 해석 (`Spark UI`, `localhost:4040`, `DAG 시각화`, `Stage`, `Task`, `GC 시간`)

- **자주 겪는 에러**
    - **[[Spark_Common_Mistakes]]** : 초보자가 자주 범하는 문법 실수 모음 (`Action 없는 실행`, `collect 남용`, `스키마 불일치`, `None 반환`)
    - **[[Spark_Troubleshooting_FileNotFound]]** : "Driver엔 있는데 Worker엔 없대요" (`경로 문제`, `HDFS vs 로컬`, `분산 파일 접근`, `절대경로`)
    - **[[WARN_NativeCodeLoader_Log]]** : 빨간색 경고 로그, 무시해도 되나요? (`NativeCodeLoader`, `WARN 레벨`, `무시 가능 에러`, `log4j 설정`)

---

## 실시간 데이터 처리 (Streaming)

> "데이터가 멈춰있지 않고 계속 흘러들어온다면?"

- **개념 및 구조 (Concept)**
    - **[[Spark_Streaming_Intro]]** : 스트리밍의 두 가지 얼굴 (`DStream`, `Structured Streaming`, `실시간 처리`, `Micro-batch`, `Continuous`)
    - **[[Spark_Streaming_Architecture]]** : 데이터 소스, 처리 모델, 트리거 설정 (`Source`, `Sink`, `Micro-batch`, `trigger`, `processingTime`, `once`)
    - **[[Streaming_Source_Comparison]]** : 소켓(Socket) vs 카프카(Kafka) 완벽 비교 (`Socket`, `Kafka`, `내구성`, `오프셋`, `실무 적합성`)
    - **[[Spark_Streaming_Fault_Tolerance]]** : 죽어도 살아나는 법 (`Checkpoint`, `Exactly-once`, `At-least-once`, `WAL`, `복구`)
    - **[[Spark_Streaming_Stateful_Stateless]]** : 변환의 종류 (`Stateful`, `Stateless`, `상태 저장`, `groupBy`, `mapGroupsWithState`)
    - **[[Spark_Streaming_Window_Aggregation]]** : 시간 단위 집계 (`Tumbling Window`, `Sliding Window`, `window 함수`, `집계 주기`)
    - **[[Spark_Streaming_Watermark]]** : 늦은 데이터 처리와 메모리 관리 (`withWatermark`, `Late Data`, `지연 허용 시간`, `메모리 해제`)

- **환경 설정 및 연동 (Setup & Integration)**
    - **[[Apache_Kafka_Concept]]** : 대용량 데이터 파이프라인의 심장, 카프카 기초 (`Topic`, `Partition`, `Offset`, `Producer`, `Consumer`, `Broker`)
    - **[[Spark_Kafka_Docker_Setup]]** : Kafka & Spark 환경 구축 (`docker-compose`, `Zookeeper`, `Broker`, `네트워크 설정`, `포트`)
    - **[[Spark_Streaming_Cassandra_Setup]]** : Cassandra 환경 구축 (`docker-compose`, `Keyspace`, `Table`, `CQL`, `포트 9042`)
    - **[[Spark_Streaming_Kafka_Integration]]** : 스파크로 카프카 데이터 읽고 쓰기 ⭐ (`readStream`, `writeStream`, `subscribe`, `startingOffsets`, `checkpointLocation`)
    - **[[Spark_Streaming_Code_Analysis]]** : 구조적 스트리밍과 Cassandra 연동 (`foreachBatch`, `Cassandra write`, `배치 처리`, `싱크 연동`)

- **심화 패턴: 조인 (Join Patterns)**
    - **[[Spark_Streaming_Stream_Join]]** : Stream-Stream Join (`두 스트림 조인`, `Watermark 필수`, `이벤트 타임`, `상태 유지`)
    - **[[Spark_Streaming_Outer_Join_Limits]]** : 스트림 간 Outer Join의 제약사항 (`Outer Join 제한`, `Watermark 필수`, `늦은 출력`, `NULL 처리`)
    - **[[Spark_Streaming_Static_Join_Run_Guide]]** : Stream-Static Join (`정적 테이블 조인`, `Watermark 불필요`, `실시간+고정 데이터`, `브로드캐스트`)

- **실습 프로젝트 (Hands-on)**
    - **[[Spark_Streaming_Socket_Boilerplate]]** : 만능 소켓 스트리밍 코드 (`Netcat`, `nc -lk`, `readStream`, `socket`, `보일러플레이트`)
    - **[[Spark_Streaming_JSON_ETL_Project]]** : 복잡한 JSON 데이터 실시간 변환 ⭐️ (`from_json`, `스키마 정의`, `변환`, `싱크`, `실전 ETL`)
    - **[[Spark_Streaming_Sink_Multiple]]** : 다중 출력 및 `.queryName` 활용법 (`foreachBatch`, `다중 Sink`, `queryName`, `awaitTermination`)

- **문제 해결 (Troubleshooting)**
    - **[[Spark_Streaming_Troubleshooting]]** : 결과가 안 나올 때 (`좀비 데이터`, `오프셋 초기화`, `환경 리셋`, `Watermark 미충족`)

---

## 저장소 및 포맷 (Storage & Formats)

> 효율적인 데이터 저장을 위한 기술들입니다.

- **파일 포맷**
    - **[[Spark_Data_Formats]]** : Parquet, Avro, ORC 완벽 비교 (`컬럼 기반`, `행 기반`, `압축율`, `스키마 내장`, `읽기/쓰기 성능`)
    - **[[Spark_Compression_Strategies]]** : 압축 방식 장단점 (`Gzip`, `Snappy`, `ZSTD`, `압축률 vs 속도`, `분할 가능 여부`)

- **Apache Iceberg**
    - **[[Apache_Iceberg_Intro]]** : 차세대 테이블 포맷 Iceberg란? (`테이블 포맷`, `ACID`, `Schema Evolution`, `Time Travel`, `Delta Lake 비교`)
    - **[[Apache_Iceberg_Setup]]** : Iceberg 로컬 실습 및 Time Travel (`로컬 설정`, `snapshot`, `롤백`, `버전 관리`, `Catalog`)

---

## 참고 자료 (Resources & Templates)

> 자주 쓰는 코드는 복사해서 쓰세요.

- **[[Spark_Kafka_Stateful_Agg_Template]]** : Kafka + Stateful 집계(GroupBy) 기본 파이썬 코드 (`GroupBy`, `상태 집계`, `Watermark`, `보일러플레이트`)
- **[[Spark_Kafka_Tumbling_Window_Template]]** : Kafka + 윈도우(시간) 집계 템플릿 (`Tumbling Window`, `window 함수`, `시간 집계`, `Kafka 연동`)
- [[Kafka_Spark_CLI_Cheatsheet]] : 토픽 생성, 스파크 실행 명령어 모음 (`kafka-topics.sh`, `spark-submit`, `CLI`, `자주 쓰는 명령어`)
- **[[Spark_Functions_Library]]** : 자주 쓰는 내장 함수 모음 (`F.col`, `F.when`, `F.explode`, `F.struct`, `함수 레퍼런스`)

> 스트리밍-정적 조인은 실시간 스트림에 변화가 적은 기존 데이터를 붙일 때 유용합니다. 스트리밍-스트리밍 조인보다 간단하며 워터마크가 필수는 아닙니다.