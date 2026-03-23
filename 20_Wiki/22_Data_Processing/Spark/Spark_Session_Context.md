---
aliases:
  - SparkSession
  - SparkContext
  - sc
  - spark
  - RDD생성
  - 세션설정
  - parallelize
  - textFile
  - setLogLevel
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Architecture]]"
  - "[[Spark_Concept_Evolution]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step1.ipynb
---
# Spark_Session_Context — SparkSession vs SparkContext

## 한 줄 요약

```
SparkSession (spark)  클러스터로 들어가는 문 — DataFrame / SQL 담당
SparkContext (sc)     엔진 직접 제어 — RDD / 저수준 기능 담당

평소엔 spark 만 쓰고 sc 는 RDD 필요할 때만 꺼냄
```

---

---

# ① spark vs sc — 핵심 차이

|항목|`spark` (SparkSession)|`sc` (SparkContext)|
|---|---|---|
|역할|DataFrame / SQL 담당|RDD / 저수준 제어|
|상하관계|전체 관리자|spark 안에 포함|
|코드|`spark.read.csv(...)`|`sc.parallelize(...)`|
|언제|**평소엔 spark 만**|RDD 필요할 때만|

```
SparkSession 이 SparkContext 를 포함하는 구조
spark = 자동차 전체
sc    = 자동차 엔진
```

---

---

# ② SparkSession 생성 — 권장 방식

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("HospitalERConsumer")        # 앱 이름
    .config("spark.driver.memory", "2g") # 설정
    .getOrCreate()                        # 있으면 가져오고 없으면 생성
)

# SparkContext 가져오기 (RDD 필요 시)
sc = spark.sparkContext

# 버전 확인
print(spark.version)   # 3.5.1
```

```
getOrCreate() 이유:
  이미 실행 중인 세션 있으면 새로 안 만들고 가져옴
  SparkContext 는 프로그램 안에 딱 하나만 존재 가능
  새로 만들려고 하면 충돌 → getOrCreate() 로 안전하게
```

---
---
# .config() — 자주 쓰는 설정


```python
spark = (
    SparkSession.builder
    .appName("HospitalERConsumer")
    # 아래처럼 .config() 를 여러 번 체이닝 가능
    .config("spark.driver.memory", "2g")
    .config("spark.executor.memory", "2g")
    .config("spark.executor.cores", "2")
    .getOrCreate()
)
```

## 메모리 설정

|설정|설명|예시|
|---|---|---|
|`spark.driver.memory`|Driver 메모리|`"2g"`|
|`spark.executor.memory`|Executor 메모리|`"4g"`|
|`spark.driver.maxResultSize`|collect() 결과 최대 크기|`"1g"`|

```
driver.memory:
  collect() 결과 / 브로드캐스트 변수 등이 Driver 메모리 사용
  너무 작으면 OOM → 기본값 1g → 보통 2~4g 로 설정

executor.memory:
  실제 데이터 처리하는 Executor 메모리
  데이터 많으면 늘려야 Spill to Disk 안 생김
```

### ⚠️ Worker Lost 에러 — 메모리 연관 설정 ⭐️

```
증상:
  ERROR TaskSchedulerImpl: Lost executor 0 on 172.18.0.7:
  worker lost: Not receiving heartbeat for 60 seconds

원인: Spark Worker 메모리 부족 → 응답 중단 → Executor 연결 끊김

수정할 곳 2가지:
  1. docker-compose.yml — SPARK_WORKER_MEMORY 증가
  2. spark-submit — --executor-memory 옵션 추가
```

```yaml
# docker-compose.yml
spark-worker:
  environment:
    - SPARK_WORKER_MEMORY=2G   # 1G → 2G
    - SPARK_WORKER_CORES=2
```

```bash
# spark-submit
spark-submit \
  --master spark://spark-master:7077 \
  --executor-memory 1g \   # 기본 512m → 1g
  --driver-memory 1g \
  consumer.py
```

```
SPARK_WORKER_MEMORY >= --executor-memory 조건 필수
Worker 가 Executor 보다 커야 할당 가능

.config() 와 --executor-memory 의 차이:
  .config("spark.executor.memory", "1g")  코드 안에서 설정
  --executor-memory 1g                    spark-submit 실행 시 설정
  spark-submit 옵션이 .config() 보다 우선 적용
```

> Docker Compose 메모리 설정 상세 → [[Docker_Compose_Setup#리소스 설정 — Spark Worker Lost 에러]] 참고

## Executor 자원 설정

|설정|설명|예시|
|---|---|---|
|`spark.executor.cores`|Executor 당 코어 수|`"2"`|
|`spark.executor.instances`|Executor 개수|`"3"`|

```
총 병렬 처리 수 = executor.cores × executor.instances
  cores=2, instances=3 → 동시에 6개 Task 처리
```

## 패키지 / JAR 설정

|설정|설명|예시|
|---|---|---|
|`spark.jars.packages`|Maven 패키지 자동 다운로드|Kafka / PostgreSQL 드라이버|
|`spark.jars`|로컬 JAR 직접 지정|`"/path/to/file.jar"`|


```python
# Kafka + PostgreSQL 연동 시
.config("spark.jars.packages",
        "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0,"
        "org.postgresql:postgresql:42.6.0")
```

## Shuffle / 성능 설정

|설정|설명|기본값|
|---|---|---|
|`spark.sql.shuffle.partitions`|Shuffle 후 파티션 수|`200`|
|`spark.default.parallelism`|기본 병렬 처리 수|코어 수|

python

```python
# 소규모 데이터에서 파티션 200개는 너무 많음 → 줄이기
.config("spark.sql.shuffle.partitions", "10")
```

```
shuffle.partitions 기본값 200 의 문제:
  join / groupBy 후 파티션이 200개로 쪼개짐
  데이터가 작으면 빈 파티션 수두룩 → 오버헤드
  → 데이터 크기에 맞게 조정 (소규모: 10~20)
```

## 로그 설정

```python
# .config() 로도 설정 가능
.config("spark.logConf", "true")  # 설정값 로그 출력

# 또는 생성 후
spark.sparkContext.setLogLevel("WARN")  # 더 자주 씀
```

---

---

# ③ sc 가 필요한 3가지 상황

## parallelize — 리스트를 RDD 로 변환

```python
data = [1, 2, 3, 4, 5]

# ❌ spark 에는 parallelize 없음
spark.parallelize(data)   # AttributeError

# ✅ sc 에서만 사용 가능
sc = spark.sparkContext
rdd = sc.parallelize(data)
```

## textFile — 텍스트 파일을 RDD 로 읽기

```python
# spark.read.text() → DataFrame (행/열 구조)
# sc.textFile()     → RDD (줄글 덩어리)

rdd = sc.textFile("logs/app.log")
# 로그 파일 등 비정형 데이터 처리 시 유용
```

## broadcast — 모든 Executor 에 변수 공유

```python
# 큰 딕셔너리를 모든 Executor 에 복사해서 보낼 때
hospital_dict = {"A1100001": "서울대병원", ...}
bc = sc.broadcast(hospital_dict)

# Executor 에서
bc.value["A1100001"]   # "서울대병원"
```

---

---

# ④ 구식 방식 — 쓰지 말 것

```python
# ❌ 구식 — sc 직접 생성
import pyspark
sc = pyspark.SparkContext('local[*]')
# → DataFrame / SQL 기능 없음
# → 이후 SparkSession 생성하면 충돌

# ✅ 현대적 방식 — SparkSession 으로 시작
spark = SparkSession.builder.getOrCreate()
sc = spark.sparkContext   # 필요하면 여기서 꺼냄
```

---

---

# ⑤ 실전 패턴 

```python
from pyspark.sql import SparkSession

# Kafka + PostgreSQL 패키지 포함
spark = (
    SparkSession.builder
    .appName("HospitalERConsumer")
    .config("spark.jars.packages",
            "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0,"
            "org.postgresql:postgresql:42.6.0")
    .getOrCreate()
)
spark.sparkContext.setLogLevel("WARN")  # 불필요한 로그 줄이기

# Kafka 에서 읽기 (DataFrame API → sc 불필요)
df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "er-realtime") \
    .load()

# 작업 끝나면 자원 반납
spark.stop()
```

---
---
# setLogLevel — 로그 레벨 설정

```
Spark 는 기본적으로 INFO 레벨로 로그를 쏟아냄
→ 실제 출력 결과가 로그에 묻혀서 안 보임
→ setLogLevel("WARN") 으로 불필요한 로그 줄이기
```


```python
spark.sparkContext.setLogLevel("레벨")
```

|레벨|의미|출력 내용|
|---|---|---|
|`ALL`|전부 출력|모든 로그|
|`DEBUG`|디버그|내부 동작 상세|
|`INFO`|정보|기본값 — 작업 진행 상황 (많음)|
|`WARN`|경고|경고 + 에러만 ← 실무 권장|
|`ERROR`|에러|에러만|
|`OFF`|끔|아무것도 안 나옴|


```python
# 개발 중 — WARN 으로 줄이기
spark.sparkContext.setLogLevel("WARN")

# 디버깅 시 — INFO 로 다시 늘리기
spark.sparkContext.setLogLevel("INFO")

# 테스트 시 — ERROR 만 보기
spark.sparkContext.setLogLevel("ERROR")
```

```
실무 관행:
  개발 / 운영: WARN  ← 경고 이상만 보임
  문제 발생 시: INFO  ← 어디서 막혔는지 추적
  성능 튜닝 시: DEBUG ← 내부 실행 계획까지 확인

WARN_NativeCodeLoader 같은 경고:
  Hadoop 압축 라이브러리 관련 경고
  실습엔 지장 없음 → WARN 으로도 뜸 → 무시해도 됨
```


---

---

# 한눈에 정리

```
SparkSession (spark):
  클러스터 연결 창구
  DataFrame / SQL / Streaming 전부 담당
  → 항상 spark 로 시작

SparkContext (sc):
  spark 안에 숨어있는 엔진
  sc = spark.sparkContext 로 꺼냄
  → parallelize / textFile / broadcast 필요할 때만

getOrCreate():
  SparkContext 는 하나만 가능
  이미 있으면 가져오고 없으면 생성 → 안전
```