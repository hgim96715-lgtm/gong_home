---
aliases:
  - Spark Home
  - 스파크 공부 지도
  - Index
tags:
  - Index
  - Spark
---

---

### 📂 [바로가기]

- **노트북 경로:** `/Users/gong/gong_study_de/apache-spark/notebooks`
- 👉 [폴더 바로 열기](file:///Users/gong/gong_study_de/apache-spark/notebooks)
- **다이어그램:** **[[01_Spark_Master_Map]]**

---

##  아키텍처 및 핵심 원리 (Architecture & Internals)

> 스파크가 **"어떻게, 그리고 왜"** 이렇게 동작하는지 설명합니다.

- **기초 개념**
    - **[[Spark_Concept_Evolution]]** : 하둡(MapReduce)과 스파크의 차이점 (Disk vs Memory)
    - **[[Spark_Architecture]]** : Driver, Executor, Cluster Manager의 역할 완벽 정리 
    - **[[RDD_Concept]]** : 스파크 데이터의 불변성과 복구 능력 (Resilient Distributed Dataset)
    - **[[Spark_Job_Scheduling]]** : 자원 배분 전쟁 (Static vs Dynamic, FIFO vs FAIR)

- **실행 엔진 (Engine)**
    - **[[Transformations_vs_Actions]]** : Lazy Evaluation과 의존성 (Narrow vs Wide, 셔플의 원리) 
    - **[[Spark_Catalyst_Optimizer]]** : 스파크의 뇌, 논리적 계획이 물리적 계획으로 바뀌는 과정
    - **[[Spark_Memory_Management]]** : Executor 메모리 구조 (Unified Memory, Storage vs Execution)

---

## 데이터 엔지니어링 (Data Engineering)

> 데이터를 **읽고, 쓰고, 변환하는** 실무 기술들을 모았습니다.

- **설정 및 시작**
    - **[[Spark_Installation_Local]]** : 로컬/Docker 환경 구축
    - **[[PySpark_Session_Context]]** : `SparkSession` 객체 생성과 필수 설정
    - **[[Spark_Session_Deep_Dive]]** : `spark-submit`, Master, Deploy Mode 심층 분석

- **DataFrame & SQL (메인 무기)**
    - **[[Spark_DataFrame_Basics]]** : 기본 조작 (Read, Show, Schema)
    - **[[Spark_Data_IO]]** : 저장의 기술 (Parquet, CSV, PartitionBy,Write)
    - **[[DataFrame_Transform_Basic]]** : 조회와 필터링 (Select, Filter, Column, withColumn, cast)
    - **[[DataFrame_Aggregation]]** : 집계와 순위 (GroupBy, Agg, Sort)
    - **[[Spark_DataFrame_Joins]]** : 조인 전략 총정리 (Inner, Left, Semi, Anti)
    - **[[Spark_Data_Cleaning]]** : 결측치와 날짜 처리 (`na.drop`, `format_number`)
    - **[[SQL_with_Spark]]** : Python 대신 SQL 문법 사용하기 (`TempView`)
    - **[[Spark_Functions_Library]]** : 자주 쓰는 내장 함수 사전 (Reference)
    - **[[Spark_JSON_Handling]]** : JSON 데이터 파싱과 생성 (from_json, to_json)

- **RDD 로우 레벨 제어 (Low-Level)**
    - **[[Spark_General_Transformations]]** : 기본 변환 (`map`, `flatMap`, `filter`) 
    - **[[Spark_Key_Transformations]]** : 키(Key) 기준 제어 (`reduceByKey`, `groupByKey`, `sortByKey`)
    - **[[Spark_Value_Transformations]]** : 값(Value)만 안전하게 변경 (`mapValues`)

- **외부 데이터 연동 (External Data Sources)** 
	- **[[Spark_JDBC_Postgres_Guide]]** : **(필수)** PostgreSQL 연동 완벽 가이드 (서버 vs 드라이버 개념, Docker 설정, JDBC 템플릿)

---

##  성능 최적화 (Optimization & Tuning)

> **"왜 느리지?"** 싶을 때 열어보는 튜닝 비법서입니다.

- **파티셔닝 (Partitioning)**
    - **[[Spark_Partitioning_Concept]]** : 파티셔닝 기초 (`repartition` vs `coalesce`)
    - **[[Spark_AQE_Deep_Dive]]** : 실행 중에 계획을 바꾸는 AQE (Skew Join 해결)
    - **[[Spark_Dynamic_Partition_Pruning(DPP)]]** : 조인할 때 필요한 데이터만 읽는 기술

- **전략 및 테크닉**
    - **[[Spark_Caching_Strategies]]** : 반복 계산을 막는 캐싱 (cache vs persist)
    - **[[Spark_Core_Broadcast]]** : 셔플을 없애는 기술 (Broadcast Variable & Join)
    - **[[Spark_SQL_Hints]]** : 옵티마이저의 판단을 강제로 덮어쓰기 (Hint)
    - **[[Spark_Speculative_Execution]]** : 느린 태스크(낙오자) 버리기 전략
    - **[[Spark_Accumulator]]** : 분산 환경의 전역 카운터 (집계 및 디버깅)
    - **[[Spark_Iterating_Data]]** : 반복문의 정석 (`collect` 절대 금지, `foreachPartition` 권장)

---

## 트러블 슈팅 및 모니터링 (Troubleshooting)

> 에러 로그를 해석하고 해결하는 방법입니다.

- **분석 도구**
    - **[[Spark_UI_Guide]]** : **(필수)** 스파크의 MRI, Event Timeline 색깔별 의미 해석

- **자주 겪는 에러**
    - **[[Spark_Common_Mistakes]]** : 초보자가 자주 범하는 문법 실수 모음
    - **[[Spark_Troubleshooting_FileNotFound]]** : "Driver엔 있는데 Worker엔 없대요" (경로 문제)
    - **[[WARN_NativeCodeLoader_Log]]** : "빨간색 경고 로그, 무시해도 되나요?"

---

##  실시간 데이터 처리 (Streaming)

> "데이터가 멈춰있지 않고 계속 흘러들어온다면?"

- **개념 및 구조 (Concept)**
    - **[[Spark_Streaming_Intro]]** : 스트리밍의 두 가지 얼굴 (DStream vs Structured Streaming)
    - **[[Spark_Streaming_Architecture]]** : 데이터 소스, 처리 모델(Micro-batch), 트리거 설정
    - **[[Streaming_Source_Comparison]]** : 소켓(Socket) vs 카프카(Kafka) 완벽 비교
    - **[[Spark_Streaming_Fault_Tolerance]]** : 죽어도 살아나는 법 (Checkpoint & Exactly-once)
    - **[[Spark_Streaming_Stateful_Stateless]]** : 변환의 종류 (기억력 유무, Stateful vs Stateless)
    - **[[Spark_Streaming_Window_Aggregation]]** : 시간 단위 집계 (Tumbling vs Sliding)
    - **[[Spark_Streaming_Watermark]]** : 늦은 데이터 처리와 메모리 관리 (Late Data Dropping)
        
- **환경 설정 및 연동 (Setup & Integration)**
    - **[[Apache_Kafka_Concept]]** : 대용량 데이터 파이프라인의 심장, 카프카 기초
    - **[[Spark_Kafka_Docker_Setup]]** : Kafka & Spark 환경 구축 (Docker Compose)
    - **[[Spark_Streaming_Cassandra_Setup]]** : Cassandra 환경 구축 (Docker Compose)
    - **[[Spark_Streaming_Kafka_Integration]]** : [코드] 스파크로 카프카 데이터 읽고 쓰기 ⭐
    - **[[Spark_Streaming_Code_Analysis]]** : 구조적 스트리밍과 Cassandra 연동 (foreachBatch 활용)

- **심화 패턴: 조인 (Join Patterns)**
    - **[[Spark_Streaming_Stream_Join]]** : Stream-Stream Join (두 개의 실시간 데이터 조인)
    - **[[Spark_Streaming_Outer_Join_Limits]]** : 스트림 간 Outer Join의 제약사항과 특징
    - **[[Spark_Streaming_Static_Join_Run_Guide]]** : Stream-Static Join (실시간 데이터 + 고정 테이블 조인)

- **실습 프로젝트 (Hands-on)**
    - **[[Spark_Streaming_Socket_Boilerplate]]** : 만능 소켓 스트리밍 코드 (`Netcat` 등)
    - **[[Spark_Streaming_JSON_ETL_Project]]** : (실전) 복잡한 JSON 데이터 실시간 변환 ⭐️
    - **[[Spark_Streaming_Sink_Multiple]]** : 다중 출력(Sink to Multiple) 및 `.queryName` 활용법

- **문제 해결 (Troubleshooting)**
    - **[[Spark_Streaming_Troubleshooting]]** : 짝이 맞는데 결과가 안 나올 때 (좀비 데이터 & 환경 리셋)

---
## 저장소 및 포맷 (Storage & Formats)

> 효율적인 데이터 저장을 위한 기술들입니다.

- **파일 포맷**
    - **[[Spark_Data_Formats]]** : Parquet, Avro, ORC 완벽 비교
    - **[[Spark_Compression_Strategies]]** : 압축 방식 장단점 (Gzip, Snappy, ZSTD)

- **Apache Iceberg**
    - **[[Apache_Iceberg_Intro]]** : 차세대 테이블 포맷 Iceberg란?
    - **[[Apache_Iceberg_Setup]]** : Iceberg 로컬 실습 및 Time Travel


---

##  참고 자료 (Resources & Templates)

> 자주 쓰는 코드는 복사해서 쓰세요.

- **[[Spark_Kafka_Stateful_Agg_Template]]** : Kafka + Stateful 집계(GroupBy) 기본 파이썬 코드 
- **[[Spark_Kafka_Tumbling_Window_Template]]** : Kafka + 윈도우(시간) 집계 템플릿
- [[Kafka_Spark_CLI_Cheatsheet]] : 토픽 생성, 스파크 실행 명령어 모음
- **[[Spark_Functions_Library]]** : 자주 쓰는 내장 함수 모음

>스트리밍-정적 조인은 실시간 스트림에 변화가 적은 기존 데이터를 붙일 때 유용합니다. 스트리밍-스트리밍 조인보다 간단하며 워터마크가 필수는 아닙니다.