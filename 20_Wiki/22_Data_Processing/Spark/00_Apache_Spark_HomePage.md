
> **노트북 경로:** `/Users/gong/gong_study_de/apache-spark/notebooks` 👉 [폴더 바로 열기](https://claude.ai/chat/09c89072-1332-44ab-9662-f57d38b75168)

---

---

# 🏗️ 아키텍처 — Spark 가 어떻게 동작하는가

| 노트                                    | 핵심 키워드                                       |
| ------------------------------------- | -------------------------------------------- |
| [[Spark_Concept_Evolution]]           | MapReduce vs Spark, In-Memory, DAG           |
| [[Spark_Architecture]]                | Driver, Executor, Cluster Manager, 자원 배분     |
| [[RDD_Concept]]                       | 불변성, Lineage, 파티션, 복구                        |
| [[Spark_Transformations_vs_Actions]]⭐ | Lazy Evaluation, Narrow/Wide, 셔플, Action 트리거 |
| [[Spark_Catalyst_Optimizer]]          | 논리 계획 → 물리 계획, Predicate Pushdown            |

---

---

# 🔧 환경 설정

| 노트                                   | 핵심 키워드                                                         |
| ------------------------------------ | -------------------------------------------------------------- |
| [[Spark_Installation_Local_Docker]]⭐ | pip install pyspark, Docker Compose, JAVA_HOME                 |
| [[Spark_Session_Context]]⭐           | SparkSession, builder, appName, getOrCreate,setLogLevel,config |

---

---

# 📊 DataFrame — 메인 무기

| 노트                             | 핵심 키워드                                                                                     |
| ------------------------------ | ------------------------------------------------------------------------------------------ |
| [[Spark_Core_Objects]] ⭐️      | SparkSession, **Row**, Column, RDD, sparkContext                                           |
| [[Spark_DataFrame]] ⭐️         | createDataFrame, StructType, StructField, spark.read, show/types                           |
| [[Spark_DataFrame_Transform]]⭐ | select, filter, withColumn, cast, alias                                                    |
| [[DataFrame_Aggregation]]      | groupBy, agg, sort, count, sum, avg                                                        |
| [[Spark_DataFrame_Joins]]      | inner, left, semi, anti, broadcast                                                         |
| [[Spark_Data_Cleaning]]        | na.drop, na.fill, to_date, date_format                                                     |
| [[Spark_Data_IO]]              | Parquet, CSV, partitionBy, write mode                                                      |
| [[SQL_with_Spark]]             | createOrReplaceTempView, spark.sql                                                         |
| [[Spark_JSON_Handling]] ⭐      | from_json, to_json, StructType, 중첩 JSON                                                    |
| [[Spark_Functions_Library]] ⭐  | F.col, F.lit, F.when, F.coalesce, F.regexp_extract,current_timestamp/pyspark.sql.functions |

---

---

# 🔩 RDD — 저수준 제어

> DataFrame 으로 해결 안 될 때 쓴다. 비정형 데이터 파싱, 커스텀 로직.

|노트|핵심 키워드|
|---|---|
|[[Spark_General_Transformations]]|map, flatMap, filter, distinct, union, Lazy|
|[[Spark_Key_Value_Transformations]]|reduceByKey, groupByKey, mapValues, PairRDD, 셔플 발생 여부|

> `Spark_Key_Transformations` + `Spark_Value_Transformations` → 합쳐서 [[Spark_Key_Value_Transformations]] 로 관리

---

---

# 🚰 Streaming — 실시간 처리

| 노트                                       | 핵심 키워드                                                  |
| ---------------------------------------- | ------------------------------------------------------- |
| [[Spark_Streaming_Intro]]                | DStream vs Structured Streaming, Micro-batch            |
| [[Spark_Streaming_Architecture]]         | Source, Sink, trigger, processingTime                   |
| [[Spark_Streaming_Kafka_Integration]] ⭐️ | readStream, writeStream, subscribe, checkpointLocation  |
| [[Spark_Streaming_Fault_Tolerance]]      | Checkpoint, Exactly-once, WAL, 복구                       |
| [[Spark_Streaming_Window_Watermark]]     | Tumbling/Sliding Window, withWatermark, 늦은 데이터 처리       |
| [[Spark_Streaming_Join_Patterns]]        | Stream-Stream Join, Stream-Static Join, Watermark 필수 여부 |
| [[Spark_Streaming_Troubleshooting]]      | 좀비 데이터, 오프셋 초기화, Watermark 미충족                          |

> `Spark_Streaming_Window_Aggregation` + `Spark_Streaming_Watermark` → [[Spark_Streaming_Window_Watermark]] 로 합침 `Spark_Streaming_Stream_Join` + `Spark_Streaming_Static_Join` → [[Spark_Streaming_Join_Patterns]] 로 합침

---

---

# 🔌 외부 연동

|노트|핵심 키워드|
|---|---|
|[[Spark_JDBC_PostgreSQL]] ⭐️|JDBC Driver .jar, 내부/외부 포트, write.jdbc|
|[[Spark_Kafka_Docker_Setup]]|docker-compose, Broker, 네트워크 설정|

---

---

# ⚡ 성능 최적화

|노트|핵심 키워드|
|---|---|
|[[Spark_Partitioning_Concept]]|repartition, coalesce, 셔플, 병렬성|
|[[Spark_Caching_Strategies]]|cache, persist, StorageLevel, unpersist|
|[[Spark_Core_Broadcast]]|Broadcast Variable, 셔플 제거, 작은 테이블|
|[[Spark_Memory_Management]]|Unified Memory, OOM, 메모리 튜닝|
|[[Spark_UI_Guide]]|localhost:4040, DAG, Stage, Task, GC 시간|

---

---

# 🗄️ 저장 포맷

|노트|핵심 키워드|
|---|---|
|[[Spark_Data_Formats]]|Parquet vs Avro vs ORC, 컬럼 기반 vs 행 기반|
|[[Spark_Compression_Strategies]]|Gzip, Snappy, ZSTD, 압축률 vs 속도|
|[[Apache_Iceberg_Intro]]|ACID, Schema Evolution, Time Travel|

---

---

# 📎 치트시트 / 템플릿

| 노트                                    | 설명                                   |
| ------------------------------------- | ------------------------------------ |
| [[Spark_Kafka_Stateful_Agg_Template]] | Kafka + GroupBy 집계 보일러플레이트           |
| [[Kafka_Spark_CLI_Cheatsheet]]        | kafka-topics.sh, spark-submit 명령어 모음 |
| [[Spark_Common_Mistakes]]             | Action 없는 실행, collect 남용, None 반환    |