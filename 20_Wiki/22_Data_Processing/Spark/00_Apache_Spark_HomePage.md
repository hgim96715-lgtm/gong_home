[[01_Spark_Master_Map]]

> [!example] **🚀 내 실습 공간 (My Workspace)**
 > 이론 공부하다가 실습이 필요하면 바로 여기로 이동! 
 >  **📂 노트북 경로:** `/Users/gong/gong_study_de/apache-spark/notebooks` 
 >   **👉 [폴더 바로 열기](file:///Users/gong/gong_study_de/apache-spark/notebooks)** > *(클릭이 안 되면 경로를 복사해서 Finder에서 `Cmd + Shift + G`로 이동하세요)*

## Level 0. 개념 잡기 (Why Spark?)

> "하둡(Hadoop)은 왜 느렸고, 스파크는 왜 빠른가?"

- [x] **[[Spark_Concept_Evolution]]** : MapReduce vs Spark (디스크 vs 메모리)
- [x] **[[Spark_Architecture]]** : Driver, Executor, Cluster Manager의 역할 분담
- [x] [[RDD_Concept]] :스파크 데이터의 최소 단위 (불변성과 복구 능력)
- [x] **[[Spark_Installation_Local]]** : 내 컴퓨터(Docker)에 PySpark 설치하기

## Level 1. 데이터 다루기 (DataFrame API)

> "SQL처럼 데이터를 조작해보자. (RDD는 몰라도 DataFrame은 필수!)"

- [x] [[PySpark_Session_Context]]: 스파크의 시작점, `SparkSession` 객체 만들기
- [x] [[Spark_Session_Deep_Dive]] :  Spark Session 전체 설정 내용(`spark-submit`, `Master`, `Deploy Mode`)
- [x] [[Spark_DataFrame_SQL_Intro]] : RDD보다 훨씬 편한 DataFrame과 SparkSQL 소개
- [x] [[Spark_DataFrame_Basics]] : 데이터 읽기(`read`), 만들기(`createDataFrame`), 확인(`show`, `printSchema`)
- [x] [[Spark_Data_IO]] : 데이터 저장의 비밀 (`read` vs `write`, 왜 폴더로 저장될까?)
- [x] [[DataFrame_Transform_Basic]] : **[1부]** 변환과 정제 (Select, Filter, Column, withColumn)
- [x] [[Spark_Data_Cleaning]] : [1.5부] 결측치 청소와 날짜 다루기 (`na.drop`, `na.fill`, `year`, `format_number`)
- [x] [[DataFrame_Aggregation]] : **[2부]** 집계와 순위 (GroupBy, Agg, Sort/orderBy)
- [x] [[Spark_DataFrame_Joins]] : [3부] 데이터 병합하기 (Inner, Left, Semi, Anti Join)
- [x] [[Spark_Functions_Library]] : 자주 쓰는 함수 사전 (Reference)
- [x] [[SQL_with_Spark]] : 파이썬 코드 대신 SQL 문법으로 쿼리하기 (`createOrReplaceTempView`)



## Level 2. 스파크의 작동 원리 (Internals) ⭐️ 핵심

> "코드를 짰는데 왜 바로 실행이 안 되지?"

- [x] [[Spark_File_IO_Basic]] :파일 읽어오기 (`textFile`, `file://` 경로의 비밀)
- [x] **[[Transformations_vs_Actions]]** :Lazy Evaluation과 의존성 (Narrow vs Wide, 셔플의 발생 원리) 
- [x] [[Spark_Catalyst_Optimizer]] : 스파크의 뇌, Logical vs Physical Plan의 변환 과정
- [x] [[Spark_Memory_Management]] : Executor 메모리 구조 (Unified Memory, Storage vs Execution)
- [x] [[Spark_General_Transformations]] : Narrow (`map`, `flatMap` ,`glom`,`sample`,`filter`)
- [x] [[Spark_Key_Transformations]]: 키 기준으로 뭉치고 정렬하기 (`reduceByKey`, `groupByKey`, `sortByKey`, `keys`,`countByKey`)
- [x] [[Spark_Value_Transformations]] : 값만 안전하게 바꾸기 (`mapValues`, `values`,`countByValue`,`flatMapValues`)
- [x] [[Spark_Iterating_Data]] : 반복문의 함정 (`collect` vs `foreach`)

## Level 3. 성능 최적화 (Optimization)

> "데이터가 100GB가 넘어가니 에러가 나요. 튜닝이 필요해!"

- [x] **[[Spark_Partitioning_Concept]]** : 데이터를 몇 조각으로 나눌 것인가? (`repartition` vs `coalesce`,`spark.range()`,`RepartitionByRange`,`getNumPartitions()`,`glom()`)  
- [x] [[Spark_AQE_Deep_Dive]] : 실행 중에 계획을 바꾸는 AQE (Coalesce, Skew Join)
- [x] [[Spark_Dynamic_Partition_Pruning(DPP)]] : 조인할 때 필요한 파티션만 골라 읽기 (DPP)
- [x] [[Spark_Caching_Strategies]] : 반복 계산을 막는 캐싱 전략 (cache vs persist)
- [x] [[Spark_SQL_Hints]] : 옵티마이저의 판단을 강제로 덮어쓰는 기술 (Hint)
- [ ] [[Spark_Accumulator]] : 분산 환경의 전역 카운터 (집계 및 디버깅)
- [x] [[Spark_Core_Broadcast]] : 셔플 제거의 기술 (Broadcast Join vs Python Dict Lookup),`broadcast`,`udf`

## Level 4. 데이터 저장과 포맷 (Storage)

> "CSV는 이제 그만. 프로들의 포맷을 쓰자."

## Level 5. 고급 주제 (Advanced)

- [[Spark_Graph_Analysis_Basic]] : 관계 데이터 분석하기 (`collect_set`, `concat_ws`, `size`)


## Level 6. 트러블 슈팅 (Troubleshooting)

> "에러가 났을 때 당황하지 않고 해결책을 찾아요."

- [[Spark_Common_Mistakes]] : "왜 안 되지?" 자주 범하는 문법 실수 모음 (Syntax Trap)
- [[Spark_Troubleshooting_FileNotFound]] : "Driver엔 파일이 있는데 Worker엔 없대요!" (Py4JJavaError)
- [[WARN_NativeCodeLoader_Log]] : "빨간색 경고 로그(NativeCodeLoader)가 뜨는데 무시해도 되나요?"