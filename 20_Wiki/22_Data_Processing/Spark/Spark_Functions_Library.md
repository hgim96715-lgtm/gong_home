---
aliases:
  - Functions
  - SparkFunctions
  - pyspark.sql.functions
  - 스파크함수
  - lit
  - when
  - round
  - rand
  - split
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_JSON_Handling]]"
  - "[[Spark_DataFrame_Transform]]"
linked: file:///Users/gong/gong_study_de/apache-spark/notebooks/step11.ipynb
---

# Spark_Functions_Library — pyspark.sql.functions

## 한 줄 요약

```
pyspark.sql.functions 모듈의 필수 함수 모음
모든 예제는 아래 import 가 되어있다고 가정

from pyspark.sql import functions as F
from pyspark.sql.functions import col, lit, when, current_timestamp ...
```

---

---

# ① 기본 — 컬럼 선택 / 상수

## F.col() — 컬럼 선택

```python
F.col("age")          # 컬럼 객체로 만들기
F.col("age") + 1      # 연산 가능
F.col("age").alias("나이")  # 별명 가능
F.col("age").desc()   # 정렬 가능

# 문자열 "age" 도 되지만
# 연산 / alias / desc 쓰려면 F.col() 필요
df.select(F.col("age") + 1)
```

## F.lit() — 상수를 컬럼으로

```python
# 모든 행에 같은 값 넣기
df.withColumn("country", F.lit("Korea"))

# None (빈 컬럼) 만들기
df.withColumn("empty", F.lit(None))
```

```
⚠️ None 을 직접 쓰면 에러
  df.withColumn("col", None)    ❌ TypeError
  df.withColumn("col", F.lit(None))  ✅

Union 시 컬럼 수 맞추기 위해 빈 컬럼 생성할 때 자주 씀
```

---

---

# ② 조건문

## F.when().otherwise() — IF / CASE WHEN

```python
df.withColumn("status",
    F.when(F.col("hvec") > 10, "여유")
     .when(F.col("hvec") > 0,  "부족")
     .otherwise("포화")          # hvec <= 0
)
```

## F.coalesce() — 첫 번째 NULL 아닌 값

```python
# phone1 없으면 phone2, 그것도 없으면 "없음"
df.select(
    F.coalesce(F.col("phone1"), F.col("phone2"), F.lit("없음"))
)
```

---

---

# ③ 날짜 / 시간

## current_timestamp() — 현재 시각 (Executor 실행) ⭐️

```python
from pyspark.sql.functions import current_timestamp

df.withColumn("created_at", current_timestamp())
```

```
⚠️ datetime.now() 와 다름

  datetime.now()         Python 함수 → Driver 에서 실행 → str
  current_timestamp()    Spark 함수  → Executor 에서 실행 → TimestampType

  DB created_at 컬럼이 TIMESTAMP 면 current_timestamp() 써야 타입 일치
  .withColumn("created_at", col("created_at").cast("timestamp"))
  또는 withColumn("created_at", current_timestamp()) 사용
```

## from_utc_timestamp() — UTC → 시간대 변환 ⭐️


```python
from pyspark.sql.functions import from_utc_timestamp, current_timestamp

df.withColumn("created_at", from_utc_timestamp(current_timestamp(), "Asia/Seoul"))
```

```
current_timestamp() 는 UTC 기준으로 반환
Spark 컨테이너 서버는 UTC 로 설정된 경우가 많음
→ DB 에 저장하면 한국 시각보다 9시간 느린 시각이 들어감

from_utc_timestamp(타임스탬프, 시간대):
  UTC 시각을 지정한 시간대 기준으로 변환
  "Asia/Seoul" = KST (+9시간)
  → DB 에 한국 실제 시각으로 저장됨
```


```python
# 예시
# current_timestamp()              = 2026-03-23 03:00:00 (UTC)
# from_utc_timestamp(..., "Asia/Seoul") = 2026-03-23 12:00:00 (KST)

# Hospital Consumer 실적용 패턴
df.withColumn("created_at", from_utc_timestamp(current_timestamp(), "Asia/Seoul"))
```

```
from_utc_timestamp vs to_utc_timestamp:
  from_utc_timestamp(ts, tz)  UTC → 해당 시간대로 변환  (DB 저장 시)
  to_utc_timestamp(ts, tz)    해당 시간대 → UTC 로 변환  (UTC 로 정규화 시)
```

## F.current_date() — 오늘 날짜

```python
df.withColumn("today", F.current_date())   # DateType
```

## F.to_date() — 문자열 → 날짜 타입

```python
# "20241225" → 2024-12-25 (DateType)
df.select(F.to_date("date_str", "yyyyMMdd"))
```

## F.date_add() / F.datediff()

```python
F.date_add(F.current_date(), 7)                    # 7일 후
F.datediff(F.col("end_date"), F.col("start_date")) # 날짜 차이 (일)
```

---

---

# ④ 문자열

## F.substring() — 자르기

```python
# 시작 인덱스 1부터 시작 (SQL 표준, Python 의 0 아님!)
F.substring("date_str", 1, 4)   # "20241225" → "2024"
```

## F.split() — 분리 → 배열

```python
F.split("fruits", ",")   # "apple,banana" → ["apple", "banana"]
```

## F.concat() — 합치기

```python
F.concat(F.col("city"), F.lit("_"), F.col("country"))
# "Seoul", "Korea" → "Seoul_Korea"
```

## F.regexp_replace() — 정규표현식 치환

```python
F.regexp_replace("phone", "-", "")   # 전화번호 하이픈 제거
```

---

---

# ⑤ 숫자

## F.round() — 반올림

```python
F.round(F.col("avg_beds"), 2)   # 소수점 2자리
```

## 집계 함수 (groupBy 와 함께)

```python
F.avg("hvec")             # 평균
F.sum("hvec")             # 합계
F.max("hvec")             # 최댓값
F.min("hvec")             # 최솟값
F.countDistinct("hpid")   # 중복 제거 후 개수
```

---

---

# ⑥ 배열 / 구조체

## F.explode() — 배열 → 행으로 펼치기

```python
# ["A", "B"] 한 행 → Row("A"), Row("B") 두 행
df.select(F.explode(F.col("my_array")))
```

## F.size() — 배열 길이

```python
F.size(F.col("my_array"))   # ["A", "B"] → 2
```

---

---

# ⑦ JSON 처리

## from_json() — JSON 문자열 → 구조체

```python
from pyspark.sql.functions import from_json
from pyspark.sql.types import StructType, StructField, StringType

schema = StructType([StructField("hpid", StringType())])

df.select(from_json(col("value"), schema).alias("data")) \
  .select("data.*")
```

## to_json() — 구조체 → JSON 문자열

```python
from pyspark.sql.functions import to_json, struct

# 여러 컬럼 → JSON 한 개 (Kafka 발행 시)
df.select(to_json(struct("*")).alias("value"))
```

> JSON 상세 → [[Spark_JSON_Handling]]

---

---

# 한눈에 정리


|함수|역할|
|---|---|
|`col("name")`|컬럼 선택|
|`lit(value)`|상수 → 컬럼|
|`when().otherwise()`|조건문|
|`coalesce()`|NULL 대체|
|`current_timestamp()`|현재 시각 (TimestampType / UTC 기준)|
|`from_utc_timestamp(ts, "Asia/Seoul")`|UTC → KST 변환 (한국 시각 저장 시)|
|`current_date()`|오늘 날짜|
|`to_date()`|문자열 → 날짜|
|`substring()`|문자열 자르기 (1-index)|
|`regexp_replace()`|정규식 치환|
|`from_json()`|JSON → 구조체|
|`to_json()`|구조체 → JSON|
|`explode()`|배열 → 행|