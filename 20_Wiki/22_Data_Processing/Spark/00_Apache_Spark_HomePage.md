

## Level 0. 개념 잡기 (Why Spark?)

> "하둡(Hadoop)은 왜 느렸고, 스파크는 왜 빠른가?"

- [x] **[[Spark_Concept_Evolution]]** : MapReduce vs Spark (디스크 vs 메모리)
- [x] **[[Spark_Architecture]]** : Driver, Executor, Cluster Manager의 역할 분담
- [x] [[RDD_Concept]] :스파크 데이터의 최소 단위 (불변성과 복구 능력)
- [x] **[[Spark_Installation_Local]]** : 내 컴퓨터(Docker)에 PySpark 설치하기

## Level 1. 데이터 다루기 (DataFrame API)

> "SQL처럼 데이터를 조작해보자. (RDD는 몰라도 DataFrame은 필수!)"

- [x] **[[PySpark_Session_Context]]** : 스파크의 시작점, `SparkSession` 객체 만들기
- [x] [[Spark_Session_Deep_Dive]] :  Spark Session 전체 설정 내용
- [ ] **[[DataFrame_Basics]]** : 데이터 읽기, 보기, 스키마 확인 (`read`, `show`, `printSchema`)
- [ ] **[[DataFrame_Operations]]** : 선택, 필터, 집계 (`select`, `filter`, `groupBy`, `agg`)
- [ ] **[[SQL_with_Spark]]** : 파이썬 코드 대신 SQL 문법으로 쿼리하기 (`createOrReplaceTempView`)

## Level 2. 스파크의 작동 원리 (Internals) ⭐️ 핵심

> "코드를 짰는데 왜 바로 실행이 안 되지?"

- [ ] **[[Lazy_Evaluation]]** : 게으른 연산과 즉시 실행의 차이
- [x] [[Spark_File_IO_Basic]] :파일 읽어오기 (`textFile`, `file://` 경로의 비밀)
- [x] **[[Transformations_vs_Actions]]** : 족보를 만드는 함수 vs 결과를 내는 함수
- [x] [[RDD_Essential_Transformations]] : 필수 3대장 (`map` vs `flatMap`, `reduceByKey`)
- [x] [[Spark_Iterating_Data]] : 반복문의 함정 (`collect` vs `foreach`)
- [ ] **[[DAG_and_Catalyst]]** : 스파크가 실행 계획(Plan)을 최적화하는 방법
- [ ] **[[Narrow_vs_Wide_Dependency]]** : 셔플(Shuffle)이 일어나는 기준

## Level 3. 성능 최적화 (Optimization)

> "데이터가 100GB가 넘어가니 에러가 나요. 튜닝이 필요해!"

- [ ] **[[Caching_and_Persistence]]** : 자주 쓰는 데이터 메모리에 박제하기 (`cache`)
- [ ] **[[Partitioning_Concept]]** : 데이터를 몇 조각으로 나눌 것인가? (`repartition` vs `coalesce`)
- [ ] **[[Shuffle_Optimization]]** : 네트워크 비용(Shuffle) 줄이기
- [ ] **[[Broadcast_Variables]]** : 작은 테이블은 모든 노드에 뿌려라 (Join 최적화)
- [ ] [[Spark_Data_Aggregation]] : 데이터 집계와 빈도수 세기 (`countByValue`, 정렬)

## Level 4. 데이터 저장과 포맷 (Storage)

> "CSV는 이제 그만. 프로들의 포맷을 쓰자."

- [ ] **[[Parquet_and_Avro]]** : 열 지향 저장소(Columnar Storage)의 장점
    
- [ ] **[[Partitioning_on_Disk]]** : 디렉토리 구조로 데이터 나누어 저장하기 (`partitionBy`)
    
## Level 5. 고급 주제 (Advanced)

- [ ] **[[Spark_Streaming_Basics]]** : 실시간 데이터 처리 맛보기
    
- [ ] **[[Cluster_Managers]]** : Standalone vs YARN vs Kubernetes 차이점

## Level 6. 트러블 슈팅 (Troubleshooting)

> "에러가 났을 때 당황하지 않고 해결책을 찾아요."

- [[Spark_Troubleshooting_FileNotFound]] : "Driver엔 파일이 있는데 Worker엔 없대요!" (Py4JJavaError)
- [[WARN_NativeCodeLoader_Log]] : "빨간색 경고 로그(NativeCodeLoader)가 뜨는데 무시해도 되나요?"