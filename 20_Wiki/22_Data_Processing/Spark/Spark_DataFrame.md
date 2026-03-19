---
aliases:
  - createDataFrame
  - read
  - show
  - printSchema
  - schema
  - describe
  - 데이터생성
  - 데이터프레임기초
  - toDF
  - withColumnRenamed
tags:
  - Spark
related:
  - "[[Spark_Session_Context]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_DataFrame_Transform]]"
  - "[[Python_Database_Connect]]"
  - "[[PostgreSQL_Setup]]"
  - "[[SQL_DDL_Create]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step6.ipynb
---
# Spark DataFrame — 데이터프레임 만들고 확인하기

## 한 줄 요약

> **DataFrame = 스파크의 표(Table). 만드는 법 2가지(직접 생성 / 파일 읽기) + 확인하는 법.**

---

---

# ① DataFrame 이란

```
스파크의 핵심 데이터 구조.
행(Row) + 열(Column) 로 이루어진 2차원 표.

pandas DataFrame 과 비슷하지만 차이점:
  pandas  → 단일 컴퓨터 메모리에서 동작
  Spark   → 여러 서버(Executor) 에 분산되어 동작

⭐️ 불변성(Immutability):
  한 번 만들면 수정 불가 (Read-Only)
  select, filter, drop 등 모든 변형은 새로운 DataFrame 을 반환
```

```python
# ❌ 흔한 실수 — 원본은 그대로
df.drop("age")
df.show()   # age 가 그대로 있음

# ✅ 반드시 변수에 다시 담아야 함
df = df.drop("age")
```

```
불변성이 장점인 이유:
  여러 서버가 동시에 같은 데이터를 읽어도 누군가 중간에 바꿀 위험 없음
  에러가 나도 원본이 살아있어서 복구 쉬움
```

---

---

# ② 스키마 — StructType / StructField ⭐️

## 스키마란

```
DataFrame 의 설계도.
  어떤 컬럼이 있는지 (이름)
  각 컬럼의 타입이 뭔지 (StringType, IntegerType ...)
  NULL 이 들어올 수 있는지 (nullable)
```

## StructType / StructField 문법

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([           # ← 전체 스키마 (리스트로 감쌈)
    StructField("이름", 타입, nullable),   # ← 컬럼 하나
    StructField("이름", 타입, nullable),
    ...
])
```

```
StructType  → 테이블 전체의 틀  (여러 컬럼을 리스트로 묶은 것)
StructField → 컬럼 하나의 정의  (이름 / 타입 / nullable)

nullable:
  True  (기본값) → NULL 허용. 값이 없어도 OK
  False          → NULL 금지. NULL 이 들어오면 에러
```

## 실전 예시 — train_delay 스키마

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

SCHEMA_DELAY = StructType([
    StructField("run_ymd",       StringType()),    # VARCHAR(8)  → StringType
    StructField("trn_no",        StringType()),    # VARCHAR(20) → StringType
    StructField("mrnt_nm",       StringType()),    # 노선명
    StructField("dptre_stn_nm",  StringType()),    # 출발역
    StructField("arvl_stn_nm",   StringType()),    # 도착역
    StructField("plan_dep",      StringType()),    # HH:MM 문자열
    StructField("plan_arr",      StringType()),
    StructField("real_dep",      StringType()),
    StructField("real_arr",      StringType()),
    StructField("dep_delay",     IntegerType()),   # 지연 분 (정수)
    StructField("arr_delay",     IntegerType()),
    StructField("dep_status",    StringType()),    # '정시(0분)' 텍스트
    StructField("arr_status",    StringType()),
    StructField("data_type",     StringType()),
    StructField("created_at",    StringType()),    # isoformat 문자열로 받음
])
```

```
DB 타입 → Spark 타입 대응:
  VARCHAR, TEXT, CHAR  → StringType()
  INTEGER, SERIAL      → IntegerType()
  BIGINT               → LongType()
  FLOAT, REAL          → FloatType()
  DOUBLE               → DoubleType()
  BOOLEAN              → BooleanType()
  DATE                 → DateType()
  TIMESTAMP            → TimestampType()
```

## nullable 을 생략하면?

```python
StructField("run_ymd", StringType())          # nullable 생략 → True (기본값)
StructField("run_ymd", StringType(), True)    # 명시적으로 True
StructField("run_ymd", StringType(), False)   # NULL 금지
```

```
Streaming 에서는 nullable=True (기본값) 로 두는 게 안전
JSON 파싱 시 키가 없으면 NULL 이 들어오기 때문
nullable=False 로 했다가 NULL 들어오면 스트림이 에러로 죽음
```

---

---

# ③ 자주 쓰는 타입 족보

|Spark 타입|DB 타입|Python 대응|설명|
|---|---|---|---|
|`StringType()`|VARCHAR, TEXT|`str`|문자열 — 이름, 코드, 상태값|
|`IntegerType()`|INTEGER|`int`|정수 (약 ±21억)|
|`LongType()`|BIGINT|`int`|큰 정수 (21억 초과)|
|`FloatType()`|REAL|`float`|실수 (소수점 7자리)|
|`DoubleType()`|DOUBLE|`float`|정밀 실수 (소수점 15자리) — 좌표, 금액|
|`BooleanType()`|BOOLEAN|`bool`|참/거짓|
|`DateType()`|DATE|`datetime.date`|날짜만 (`2026-03-11`)|
|`TimestampType()`|TIMESTAMP|`datetime.datetime`|날짜 + 시각|

```
import 방법:
  from pyspark.sql.types import StringType, IntegerType, ...
  또는
  from pyspark.sql.types import *   (전체 한번에)
```

---

---

# ④ DataFrame 직접 만들기 — createDataFrame

## 기본 문법

```python
df = spark.createDataFrame(data, schema)
```

## 방법 1 — 리스트 + 컬럼명 (간단한 테스트용)

```python
data = [
    ("KTX001", "서울", "부산", 10),
    ("KTX002", "서울", "대전",  3),
]
columns = ["trn_no", "dptre", "arvl", "delay"]

df = spark.createDataFrame(data, schema=columns)
df.show()
```

```
+-------+----+----+-----+
| trn_no|dptre|arvl|delay|
+-------+----+----+-----+
|KTX001 |서울 |부산 |   10|
|KTX002 |서울 |대전 |    3|
+-------+----+----+-----+
```

## 방법 2 — StructType 명시 (실무 권장)

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("trn_no", StringType()),
    StructField("dptre",  StringType()),
    StructField("arvl",   StringType()),
    StructField("delay",  IntegerType()),
])

df = spark.createDataFrame(data, schema=schema)
```

```
컬럼명만 주면 타입을 Spark 가 알아서 추론
StructType 으로 명시하면 타입을 내가 확실히 제어
→ 실무에서는 StructType 을 쓰는 것이 안전
```

## 방법 3 — Row 객체 (비정형 데이터 처리용)

```python
from pyspark.sql import Row

raw_lines = [
    "KTX001|서울|부산|10",
    "KTX002|서울|대전|3",
]

def parse_line(line: str):
    fields = line.split("|")
    return Row(
        trn_no=fields[0],
        dptre=fields[1],
        arvl=fields[2],
        delay=int(fields[3]),   # 타입 변환도 여기서
    )

rdd  = spark.sparkContext.parallelize(raw_lines)
rows = rdd.map(parse_line)
df   = spark.createDataFrame(rows)
```

```
Row(키=값) 형태로 만들면
createDataFrame 할 때 키가 자동으로 컬럼 이름이 됨

언제 씀?
  로그 파일, 파이프(|) 구분 텍스트 같은 비정형 데이터를
  파이썬 함수로 한 줄씩 정제할 때
```

>→ Row 상세: [[Spark_Core_Objects#② Row]] 참고
---

---

# ⑤ 파일에서 읽기 — spark.read

## CSV

```python
# 기본
df = spark.read.csv("data.csv", header=True, inferSchema=True)

# 헤더 없는 파일 → 컬럼명 직접 지정
df = spark.read.csv("data.csv", header=False, inferSchema=True) \
           .toDF("trn_no", "dptre", "arvl", "delay")

# 스키마 미리 지정 (가장 정확)
df = spark.read.csv("data.csv", schema=my_schema, header=False)
```

```
header 옵션:
  True  → 첫 번째 줄을 컬럼명으로 사용
  False → 첫 번째 줄부터 데이터. 컬럼명은 _c0, _c1 ... 자동 생성

inferSchema 옵션:
  True  → Spark 가 타입 자동 추론 (편리하지만 느림)
  False → 전부 StringType 으로 읽음 (빠르지만 타입 변환 따로 필요)
```

## JSON / Parquet

```python
df = spark.read.json("data.json")
df = spark.read.parquet("data.parquet")   # 스키마 자동 포함, 가장 빠름
```

## JDBC (PostgreSQL)

```python
df = spark.read.jdbc(
    url="jdbc:postgresql://postgres:5432/train_db",
    table="train_delay",
    properties={
        "user":     "train_user",
        "password": "train_password",
        "driver":   "org.postgresql.Driver",
    }
)
```

> JDBC 환경 세팅 → [[Spark_JDBC_PostgreSQL]]

---

---

# ⑥ 데이터 확인 — show / printSchema / describe

## show() — 데이터 눈으로 보기

```python
df.show()                            # 기본 (상위 20행)
df.show(5)                           # 상위 5행만
df.show(truncate=False)              # 긴 값 자르지 않고 전부 출력
df.show(5, truncate=False, vertical=True)  # 5행, 전체 출력, 세로 정렬 (★ 추천)
```

```
vertical=True 로 보면:
  -RECORD 0-----------------
  trn_no     | KTX001
  dptre      | 서울
  arvl       | 부산
  delay      | 10

컬럼이 많을 때 훨씬 읽기 편함
```

## printSchema() — 타입 구조 확인

```python
df.printSchema()
# root
#  |-- trn_no: string (nullable = true)
#  |-- dptre:  string (nullable = true)
#  |-- delay:  integer (nullable = true)
```

```
언제 씀?
  스키마 정의 후 실제로 잘 적용됐는지 확인
  JSON 파싱 후 타입이 맞게 들어왔는지 확인
```

## describe() — 기초 통계

```python
df.describe().show()               # 전체 컬럼
df.describe("delay").show()        # 특정 컬럼만
```

```
출력: count / mean / stddev / min / max
숫자 컬럼에서 유용 (문자열 컬럼은 의미 없음)
```

---

---

# ⑦ show() 주의사항 — None 반환

```python
# ❌ 틀린 패턴
df2 = df.select("trn_no").show()   # show() 는 None 반환
df2.printSchema()                   # AttributeError: NoneType

# ✅ 올바른 패턴 — 변환과 출력 분리
df2 = df.select("trn_no")   # 변환 → DataFrame 반환 → 변수에 담기
df2.show()                   # 출력 → None 반환 → 변수에 담지 않기
```

```
Transformation (변환):  select, filter, withColumn, drop ...
  → 새로운 DataFrame 반환 → 변수에 담아야 함

Action (실행):  show, count, write, collect ...
  → 실제 작업 수행, None 반환 → 변수에 담으면 안 됨
```

---

---

# 치트시트

```python
# 스키마 정의
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("컬럼명", StringType()),     # nullable=True 기본값
    StructField("컬럼명", IntegerType(), False),  # NULL 금지
])

# DataFrame 생성
df = spark.createDataFrame(data, schema=schema)

# 파일 읽기
df = spark.read.csv("file.csv", header=True, inferSchema=True)
df = spark.read.parquet("file.parquet")

# 확인
df.show(5, truncate=False, vertical=True)
df.printSchema()
df.describe("컬럼명").show()

# 수정 (반드시 재할당)
df = df.drop("불필요한컬럼")
df = df.withColumn("새컬럼", ...)
```