---
aliases:
  - Spark JSON
  - from_json
  - to_json
  - json_tuple
  - get_json_object
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_DataFrame]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
  - "[[DataFrame_Transform]]"
---
# Spark JSON Handling — JSON 파싱 / 직렬화

## 한 줄 요약

> **from_json = JSON 문자열 → 컬럼들로 뜯어내기 / to_json = 컬럼들 → JSON 문자열로 포장하기**

---

---

# ① 왜 JSON 변환이 필요한가

```
Kafka 는 모든 데이터를 Binary(바이트) 로 저장한다.
Spark 가 읽으면 value 컬럼 하나에 통째로 들어온다.

{"trn_no": "KTX001", "status": "운행중", "progress_pct": 42}
                    ↑
              이게 통짜 문자열 하나

이 상태에서는 trn_no 가 뭔지, progress_pct 가 얼마인지 모른다.
각 필드를 독립적인 컬럼으로 분리해야 분석/저장이 가능하다.
→ from_json 이 하는 일
```

```
반대로 가공한 데이터를 Kafka 로 다시 보낼 때:
  여러 컬럼 → JSON 문자열 하나로 합쳐야 함 (Kafka 는 value 컬럼 하나만 받음)
→ to_json 이 하는 일
```

|함수|방향|언제 씀|
|---|---|---|
|`from_json`|JSON 문자열 → 컬럼들|Kafka 에서 읽은 데이터 파싱|
|`to_json`|컬럼들 → JSON 문자열|Kafka 로 데이터 전송|
|`get_json_object`|JSON 문자열 → 값 하나|빠르게 필드 하나만 꺼낼 때|

---

---

# ② from_json — JSON 문자열을 컬럼으로 뜯어내기

## 왜 이름이 from 인가

```
from_json = "JSON 으로부터 꺼낸다"

from_json(col("value"), schema)
           ^^^^^^^^^^   ^^^^^^
           JSON 문자열  설계도 (어떤 필드가 있는지)

JSON 문자열이라는 원재료에서
스키마를 보고 필드를 하나씩 꺼내
Struct(구조체) 로 만든다.
```

## 스키마가 왜 필수인가

```
Spark 는 value 컬럼 안에 뭐가 들어있는지 모른다.
그냥 String 으로만 보인다.

"어떤 키가 있고 타입이 뭔지" 를 스키마로 알려줘야
각 키를 독립 컬럼으로 분리할 수 있다.

스키마를 안 주면:
  → from_json 을 쓸 수 없음
  → get_json_object 로 하나씩 꺼내는 방법밖에 없음
```

## 기본 사용법

```python
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

# 1. 스키마 정의 — JSON 안에 있는 필드와 타입을 그대로 적음
schema = StructType([
    StructField("trn_no",       StringType()),
    StructField("status",       StringType()),
    StructField("progress_pct", IntegerType()),
])

# 2. from_json 적용
parsed = df.select(
    from_json(col("value"), schema).alias("data")
)
# 결과: data 컬럼 하나 — 안에 trn_no, status, progress_pct 가 중첩된 구조체

# 3. 구조체 펼치기
result = parsed.select("data.*")
# 결과: trn_no / status / progress_pct 독립 컬럼으로 분리됨
```

## value 가 Binary 일 때 — cast 먼저

```python
# Kafka 에서 바로 읽으면 value 가 Binary 상태
# from_json 은 String 만 받으므로 cast 먼저 해야 함

df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

# 또는 selectExpr 로 먼저 String 으로 변환 후 from_json
df.selectExpr("CAST(value AS STRING) AS value") \
  .select(from_json(col("value"), schema).alias("data")) \
  .select("data.*")
```

```
순서:
  Binary → cast("string") → from_json → select("data.*")

순서를 바꾸면 안 됨:
  from_json 은 StringType 컬럼만 받음
  Binary 상태로 from_json 넣으면 전부 null 로 나옴
```

---

---

# ③ from_json — 토픽마다 스키마가 달라야 하는 이유

```
Kafka 토픽이 3개 → 각 토픽이 보내는 JSON 구조가 다름
→ 토픽마다 스키마를 따로 정의해야 함
```

```python
# train-schedule 토픽 → 운행계획 JSON 구조
SCHEMA_SCHEDULE = StructType([
    StructField("run_ymd",           StringType()),
    StructField("trn_no",            StringType()),
    StructField("dptre_stn_cd",      StringType()),
    StructField("dptre_stn_nm",      StringType()),
    StructField("arvl_stn_cd",       StringType()),
    StructField("arvl_stn_nm",       StringType()),
    StructField("trn_plan_dptre_dt", StringType()),
    StructField("trn_plan_arvl_dt",  StringType()),
    StructField("data_type",         StringType()),
])

# train-realtime 토픽 → 실시간 현황 JSON 구조
SCHEMA_REALTIME = StructType([
    StructField("trn_no",       StringType()),
    StructField("mrnt_nm",      StringType()),
    StructField("dptre_stn_nm", StringType()),
    StructField("arvl_stn_nm",  StringType()),
    StructField("plan_dep",     StringType()),
    StructField("plan_arr",     StringType()),
    StructField("status",       StringType()),
    StructField("progress_pct", IntegerType()),
    StructField("data_type",    StringType()),
])

# train-delay 토픽 → 지연 분석 JSON 구조
SCHEMA_DELAY = StructType([
    StructField("run_ymd",       StringType()),
    StructField("trn_no",        StringType()),
    StructField("dep_delay",     IntegerType()),
    StructField("arr_delay",     IntegerType()),
    StructField("dep_status",    StringType()),
    StructField("arr_status",    StringType()),
    StructField("data_type",     StringType()),
    # ... 기타 필드
])
```

```
스키마를 잘못 쓰면 어떻게 되나:

  SCHEMA_SCHEDULE 으로 train-realtime 토픽 파싱 시도
  → SCHEMA_SCHEDULE 에 없는 필드(status, progress_pct)는 null 로 나옴
  → SCHEMA_SCHEDULE 에만 있는 필드(run_ymd 등)도 매칭이 안 되어 null

  각 토픽 구독 시 반드시 그 토픽에 맞는 스키마를 써야 함
```

```python
# 올바른 사용 — 토픽과 스키마가 1:1 대응
schedule_stream = (
    spark.readStream.format("kafka")
    .option("subscribe", "train-schedule")       # 토픽
    .load()
    .select(from_json(col("value").cast("string"), SCHEMA_SCHEDULE).alias("data"))  # 해당 토픽 스키마
    .select("data.*")
)

realtime_stream = (
    spark.readStream.format("kafka")
    .option("subscribe", "train-realtime")       # 다른 토픽
    .load()
    .select(from_json(col("value").cast("string"), SCHEMA_REALTIME).alias("data"))  # 다른 스키마
    .select("data.*")
)
```

---

---

# ④ to_json — 컬럼들을 JSON 문자열로 포장하기

## 기본 사용법

```python
from pyspark.sql.functions import to_json, struct, col

# 여러 컬럼 → JSON 문자열 하나
df.select(
    to_json(struct(col("trn_no"), col("status"))).alias("value")
)
# 결과: '{"trn_no": "KTX001", "status": "운행중"}'
```

## 전체 컬럼을 JSON 으로

```python
# struct("*") — 모든 컬럼을 하나의 구조체로
df.select(
    to_json(struct("*")).alias("value")
)
```

```
struct("*") 의 의미:
  "*" = 모든 컬럼
  struct 로 묶으면 컬럼들이 하나의 구조체가 됨
  to_json 으로 JSON 문자열로 변환
  .alias("value") 로 Kafka 가 받을 수 있는 컬럼명으로 지정

Kafka 로 쓸 때 이 패턴이 필수인 이유:
  Kafka writeStream 은 value 컬럼 하나만 받음
  컬럼이 여러 개인 상태로는 writeStream.format("kafka") 에러 발생
```

---

---

# ⑤ get_json_object — 필드 하나만 빠르게 꺼내기

```python
from pyspark.sql.functions import get_json_object

# $ = 루트, . = 하위 필드 접근
df.select(
    get_json_object(col("value"), "$.trn_no").alias("trn_no"),
    get_json_object(col("value"), "$.status").alias("status"),
)

# 중첩 JSON
df.select(
    get_json_object(col("value"), "$.train.info.trn_no").alias("trn_no")
)
```

```
from_json vs get_json_object:

  from_json
    → 스키마 정의 필요
    → 한 번에 모든 필드를 컬럼으로 분리
    → 반복 접근 시 빠름 (한 번 파싱으로 끝)
    → 실무 권장

  get_json_object
    → 스키마 불필요
    → 필드 하나씩 꺼냄
    → 필드마다 JSON 을 다시 파싱 → 필드가 많으면 느림
    → 빠른 확인 / 필드 1~2개만 필요할 때 적합
```

---

---

# ⑥ ArrayType — 배열 JSON

```
JSON 이 객체({}) 가 아니라 배열([]) 로 오는 경우
```

```python
from pyspark.sql.types import ArrayType, StructType, StructField, StringType

# JSON: [{"name": "KTX001"}, {"name": "KTX002"}]

item_schema = StructType([
    StructField("name", StringType()),
])

# ArrayType 으로 감싸기
array_schema = ArrayType(item_schema)

df.select(
    from_json(col("value"), array_schema).alias("trains")
)
```

---

---

# 치트시트

```python
from pyspark.sql.functions import from_json, to_json, get_json_object, struct, col
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, ArrayType

# 스키마 정의
schema = StructType([
    StructField("trn_no",  StringType()),
    StructField("delay",   IntegerType()),
])

# Kafka Binary → 컬럼들 (가장 많이 쓰는 패턴)
result = (
    df
    .select(from_json(col("value").cast("string"), schema).alias("data"))
    .select("data.*")
)

# 컬럼들 → Kafka value 컬럼
kafka_df = df.select(to_json(struct("*")).alias("value"))

# 필드 하나만 꺼내기
df.select(get_json_object(col("value"), "$.trn_no").alias("trn_no"))

# 배열 JSON
array_schema = ArrayType(schema)
df.select(from_json(col("value"), array_schema).alias("items"))
```