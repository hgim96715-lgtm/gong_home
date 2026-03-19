---
aliases:
  - DataFrame변환
  - Select
  - Filter
  - withColumn
  - 전처리
tags:
  - Spark
related:
  - "[[DataFrame_Aggregation]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_DataFrame]]"
  - "[[Spark_Functions_Library]]"
  - "[[Spark_Data_Cleaning]]"
  - "[[Spark_Streaming_JSON_ETL_Project]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
  - "[[Spark_Core_Objects]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step9.ipynb
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step7.ipynb
---
# DataFrame Transform — 조회 / 필터 / 컬럼 조작

## 한 줄 요약

> **select(열 선택) + filter(행 선택) + withColumn(열 추가/수정) — 세 가지가 핵심.**

---

---

# ① 대원칙 — 불변성

```
모든 변환은 원본을 바꾸지 않는다.
항상 새로운 DataFrame 을 반환한다.
→ 변경 결과를 쓰려면 반드시 변수에 다시 담아야 한다.
```

```python
df.filter(F.col("age") > 30)   # ❌ 결과가 사라짐 (원본 그대로)
df = df.filter(F.col("age") > 30)  # ✅ 변수에 다시 담아야 저장됨
```

---

---

# ② import — F 별명 규칙

```python
# ❌ 하나씩 가져오면 코드 상단이 지저분해짐
from pyspark.sql.functions import col, lit, sum, avg, when, expr

# ✅ F 별명으로 통째로 가져오기 (실무 표준)
import pyspark.sql.functions as F

# 사용법
F.col("age")
F.lit("Korea")
F.sum("salary")
```

---

---

# ③ 컬럼 참조 — col / 문자열 / `df[]`

```python
import pyspark.sql.functions as F

# 방법 1 — F.col() (가장 권장)
df.select(F.col("age"))

# 방법 2 — 문자열
df.select("age")

# 방법 3 — df["컬럼명"]
df.select(df["age"])
```

```
언제 F.col() 이 필요한가:

  단순 조회       → 문자열로 충분
    df.select("name", "age")

  연산 / 조건     → F.col() 필요
    F.col("age") + 1
    F.col("age") > 30
    F.col("name").alias("이름")
    F.col("age").cast("double")

  문자열 "age" 는 그냥 이름 — 연산 불가
  F.col("age") 는 연산 가능한 객체
```

---

---

# ④ select — 열 선택 / 가공

## 기본 선택

```python
# 컬럼 선택
df.select("name", "age").show()
df.select(F.col("name"), F.col("age")).show()

# 전체 컬럼 확인
print(df.columns)   # ['name', 'age', 'dept', ...]
```

## alias — 컬럼 이름 바꾸기

```python
df.select(
    F.col("name").alias("이름"),
    F.col("age").alias("나이"),
).show()
```

```
⚠️ alias 는 F.col() 객체에만 붙일 수 있다
   "age".alias("나이")     → AttributeError
   F.col("age").alias("나이") → ✅
```

## 연산 후 별명 필수

```python
# ❌ 연산하면 컬럼명이 수식 그대로 들어감
df.select(F.col("age") + 1).show()
# 컬럼명: "(age + 1)"

# ✅ alias 로 깔끔하게 정리
df.select((F.col("age") + 1).alias("korean_age")).show()
# 컬럼명: "korean_age"
```

## selectExpr — SQL 문법으로 select

```python
# F.col() 방식
df.select(
    (F.col("age") + 1).alias("korean_age"),
    F.col("name").alias("user_name"),
    F.col("salary").cast("integer"),
).show()

# selectExpr 방식 — SQL 문법 그대로 (더 간결)
df.selectExpr(
    "age + 1 as korean_age",
    "name as user_name",
    "cast(salary as integer) as salary",
).show()
```

```
selectExpr 이 특히 유용한 경우:
  계산식이 복잡할 때  → "price * qty * 0.9 as total"
  타입 변환 여러 개  → "cast(value as string)"
  CASE WHEN 쓸 때   → SQL 이 더 읽기 편함
```

---

---

# ⑤ filter / where — 행 선택

```
filter 와 where 는 완전히 동일한 함수다.
```

## 기본 사용

```python
# 문자열 방식 (SQL 스타일)
df.filter("age > 30").show()
df.where("dept = 'IT'").show()   # 문자열 비교 → 작은따옴표 사용

# F.col 방식 (Python 스타일)
df.filter(F.col("age") > 30).show()
df.filter(F.col("dept") == "IT").show()
```

## AND / OR 조건

```python
# ⚠️ F.col 방식은 각 조건을 반드시 괄호로 감싸야 함
# Python 연산자 우선순위 때문에 괄호 없으면 에러

# AND (&)
df.filter( (F.col("age") > 30) & (F.col("dept") == "IT") ).show()

# OR (|)
df.filter( (F.col("dept") == "IT") | (F.col("dept") == "HR") ).show()

# NOT (~)
df.filter( ~(F.col("dept") == "IT") ).show()

# 문자열 방식은 괄호 없어도 됨
df.filter("age > 30 AND dept = 'IT'").show()
df.filter("dept = 'IT' OR dept = 'HR'").show()
```

## NULL 처리 — isNull / isNotNull

```python
# ❌ == None 은 절대 안 됨 (에러 없이 잘못된 결과 나옴)
df.filter(F.col("email") == None)

# ✅ isNull / isNotNull 사용
df.filter(F.col("email").isNull()).show()      # NULL 인 행만
df.filter(F.col("email").isNotNull()).show()   # NULL 아닌 행만

# SQL 방식
df.filter("email IS NULL").show()
df.filter("email IS NOT NULL").show()
```

## isin — 여러 값 중 하나인지

```python
# 특정 값 목록에 포함되는지
df.filter(F.col("dept").isin("IT", "HR", "Finance")).show()

# 포함 안 되는 행만
df.filter(~F.col("dept").isin("IT", "HR")).show()

# 리스트로도 가능
target = ["IT", "HR"]
df.filter(F.col("dept").isin(target)).show()
```

## between — 범위 조건

```python
# 30 이상 40 이하 (양쪽 포함)
df.filter(F.col("age").between(30, 40)).show()

# 날짜 범위
df.filter(F.col("created_at").between("2026-01-01", "2026-03-31")).show()
```

## startswith / endswith / contains — 문자열 조건

```python
df.filter(F.col("name").startswith("K")).show()    # K 로 시작
df.filter(F.col("name").endswith("son")).show()    # son 으로 끝남
df.filter(F.col("name").contains("il")).show()     # il 포함
```

---

---

# ⑥ withColumn — 컬럼 추가 / 수정

```python
# 문법
df.withColumn("새_컬럼명", 컬럼_객체_또는_수식)
```

```
두 번째 인자는 반드시 Column 객체여야 함
숫자나 문자열 리터럴은 F.lit() 로 감싸야 함
```

## 상수값 채우기 — F.lit()

```python
# ❌ 그냥 값을 넣으면 에러
df.withColumn("country", "Korea")     # TypeError

# ✅ F.lit() 으로 감싸기
df.withColumn("country", F.lit("Korea")).show()
df.withColumn("version", F.lit(1)).show()
df.withColumn("is_active", F.lit(True)).show()
```

## 계산 / 변환

```python
# 기존 컬럼 기반 계산
df.withColumn("korean_age", F.col("age") + 1).show()
df.withColumn("total", F.col("price") * F.col("qty")).show()

# 기존 컬럼 덮어쓰기 (컬럼명 동일하게)
df = df.withColumn("age", F.col("age").cast("integer"))
```

## expr — SQL 수식으로 계산

```python
from pyspark.sql.functions import expr

# F.col 방식 대신 SQL 수식 사용 가능
df.withColumn("discount", expr("price * 0.9")).show()
df.withColumn("grade", expr("CASE WHEN score >= 90 THEN 'A' WHEN score >= 80 THEN 'B' ELSE 'C' END")).show()
```

```
selectExpr vs expr:
  selectExpr → select 전체를 SQL 로 작성
  expr       → withColumn / filter 등 특정 컬럼 하나만 SQL 수식으로
```

---

---

# ⑦ withColumnRenamed / drop — 이름 변경 / 삭제

## withColumnRenamed — 이름만 바꾸기

```python
# 단일 컬럼
df = df.withColumnRenamed("age", "user_age")

# 여러 컬럼 — 체이닝
df = (
    df
    .withColumnRenamed("age",  "user_age")
    .withColumnRenamed("name", "user_name")
)
```

```
withColumn vs withColumnRenamed:
  withColumn         → 데이터(값)를 새로 만들거나 변환  (비용 발생)
  withColumnRenamed  → 이름표만 바꿈                   (매우 빠름)

없는 컬럼 이름을 지정해도 에러 안 남 — 조용히 무시됨
```

## drop — 컬럼 삭제

```python
# 단일 컬럼 삭제
df = df.drop("불필요한컬럼")

# 여러 컬럼 삭제
df = df.drop("col1", "col2", "col3")

# 리스트로도 가능
remove_cols = ["col1", "col2"]
df = df.drop(*remove_cols)
```

```
⚠️ 불변성 — drop 도 원본을 바꾸지 않음
   df.drop("age")      → 원본 df 에는 age 가 그대로 있음
   df = df.drop("age") → 이래야 반영됨
```

---

---

# ⑧ cast — 타입 변환

```python
# withColumn 안에서
df = df.withColumn("age", F.col("age").cast("integer"))
df = df.withColumn("price", F.col("price").cast("double"))

# select 안에서
df.select(F.col("age").cast("integer")).show()

# selectExpr 로
df.selectExpr("cast(age as integer) as age").show()
```

```
자주 쓰는 타입 문자열:
  "string"     → StringType
  "integer"    → IntegerType
  "long"       → LongType
  "double"     → DoubleType
  "boolean"    → BooleanType
  "date"       → DateType
  "timestamp"  → TimestampType

Kafka value 변환 패턴 (가장 많이 씀):
  df.selectExpr("CAST(value AS STRING)")
  Binary → String → from_json 으로 파싱 가능해짐
```

## CAST 안의 AS vs 바깥의 AS

```sql
CAST(value AS STRING) AS json_str
--        ^^               ^^
--    타입 지정          컬럼명 지정
```

```
CAST 안의 AS  → "value 를 STRING 타입으로 변환해라"
               CAST 문법의 일부 — 생략 불가

바깥의 AS     → "변환된 컬럼 이름을 json_str 로 붙여라"
               alias 역할 — 생략하면 컬럼명이 수식 그대로
```


```python
# 바깥 AS 생략 vs 포함 차이

df.selectExpr("CAST(value AS STRING)").printSchema()
# |-- CAST(value AS STRING): string   ← 이름이 수식 그대로 들어감

df.selectExpr("CAST(value AS STRING) AS value").printSchema()
# |-- value: string                   ← 이름이 깔끔하게 "value"
```

```
실무에서는 뒤에 AS 를 항상 붙인다.
다음 단계에서 col("value") 로 접근해야 하는데
col("CAST(value AS STRING)") 처럼 수식 그대로 쓰면 에러 나기 쉽고
코드도 지저분해짐.
```

## Kafka value 변환 패턴

```python
# Binary → String 변환 (Kafka 필수 패턴)
df.selectExpr("CAST(value AS STRING) AS value")

# 이후 from_json 으로 JSON 파싱 가능
from pyspark.sql.functions import from_json
parsed = df.select(from_json(F.col("value"), schema).alias("data")).select("data.*")
```

```
Binary → CAST AS STRING → from_json → 컬럼들
순서가 바뀌면 안 됨
from_json 은 String 타입 컬럼만 받기 때문
```


---

---

# ⑨ struct — 여러 컬럼을 하나로 묶기

```python
from pyspark.sql.functions import struct, to_json

# 여러 컬럼 → 구조체 하나로
df.withColumn("info", struct(F.col("name"), F.col("age"))).show()
# 결과: info = {"name": "Kim", "age": 30}

# 전체 컬럼을 JSON 으로 변환 (Kafka 에 쓸 때)
df.select(to_json(struct("*")).alias("value"))
```

```
struct 를 쓰는 이유:
  to_json() 은 컬럼 하나만 받음
  여러 컬럼을 JSON 으로 만들려면 struct 로 묶어야 함

  [name="Kim", age=30] (컬럼 2개)
  → struct: {name: "Kim", age: 30} (구조체 1개)
  → to_json: '{"name": "Kim", "age": 30}' (JSON 문자열)
  → Kafka value 컬럼으로 전송 가능
```

---

---

# 함수 요약표

|함수|역할|입력 타입|
|---|---|---|
|`select()`|열 선택 / 가공|문자열 또는 Column 객체|
|`selectExpr()`|SQL 문법으로 select|SQL 문자열|
|`filter()` / `where()`|행 필터링|문자열 또는 Column 조건|
|`withColumn()`|열 추가 / 수정|Column 객체 (F.lit, F.col ...)|
|`withColumnRenamed()`|열 이름만 변경|문자열 2개|
|`drop()`|열 삭제|문자열|
|`.cast()`|타입 변환|문자열 타입명|
|`.alias()`|컬럼 별명|문자열|
|`expr()`|컬럼 하나를 SQL 수식으로|SQL 문자열|
|`struct()`|여러 컬럼 → 구조체 1개|Column 객체들|
|`.isin()`|값 목록 포함 여부|값 목록|
|`.between()`|범위 조건|시작값, 끝값|
|`.isNull()`|NULL 여부|-|
|`.isNotNull()`|NOT NULL 여부|-|

---

---

# 치트시트

```python
import pyspark.sql.functions as F
from pyspark.sql.functions import expr, struct, to_json

# 열 선택
df.select("name", "age")
df.select(F.col("age").alias("나이"))
df.selectExpr("age + 1 as korean_age", "cast(salary as int) as salary")

# 행 필터
df.filter(F.col("age") > 30)
df.filter( (F.col("age") > 30) & (F.col("dept") == "IT") )
df.filter(F.col("dept").isin("IT", "HR"))
df.filter(F.col("age").between(20, 40))
df.filter(F.col("email").isNotNull())

# 열 추가 / 수정
df.withColumn("country", F.lit("Korea"))
df.withColumn("total", F.col("price") * F.col("qty"))
df.withColumn("age", F.col("age").cast("integer"))
df.withColumn("grade", expr("CASE WHEN score >= 90 THEN 'A' ELSE 'B' END"))

# 열 이름 변경 / 삭제
df.withColumnRenamed("age", "user_age")
df.drop("col1", "col2")

# Kafka 용 JSON 변환
df.select(to_json(struct("*")).alias("value"))

# Kafka value 파싱
df.selectExpr("CAST(value AS STRING)")
```