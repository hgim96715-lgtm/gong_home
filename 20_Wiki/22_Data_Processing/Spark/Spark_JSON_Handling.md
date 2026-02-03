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
  - "[[DataFrame_Transform_Basic]]"
  - "[[Spark_Streaming_JSON_ETL_Project]]"
---
---
## 🔗 연관 개념 학습

> **실전 예제:** 이 문법들이 실제 프로젝트에서 어떻게 쓰이는지 궁금하다면?
> 👉 **[[Spark_Streaming_JSON_ETL_Project|Spark Kafka JSON 실전 프로젝트]]**

> **기초 문법:** `select`, `withColumn`, `struct` 등 기본 조작법이 헷갈린다면?
> 👉 **[[DataFrame_Transform_Basic]]**
---


## 개요: JSON을 다루는 두 가지 방향 

스파크에서 JSON 데이터 처리는 **"포장지 뜯기(Parsing)"** 와 **"다시 포장하기(Serialization)"** 두 가지로 나뉩니다.

| 함수 | 방향 | 설명 | 필수 준비물 |
| :--- | :--- | :--- | :--- |
| **`from_json`** | String ➡ Struct | JSON 문자열을 뜯어서 **컬럼(구조체)**으로 만듦 | **Schema (설계도)** |
| **`to_json`** | Struct ➡ String | 컬럼들을 묶어서 **JSON 문자열**로 변환함 | 없음 (자동) |

---
##  `from_json`: 문자열 뜯어내기 (String -> Struct) 

카프카 메시지나 로그 파일처럼 **"통짜 문자열"** 로 된 JSON을 해석해서, 우리가 사용할 수 있는 **표(Table/Struct)** 형태로 바꿉니다.

### ⚠️ 핵심 규칙: 스키마(Schema)가 필수!

컴퓨터는 멍청해서 JSON 안에 뭐가 들어있는지 미리 알려줘야 합니다.

- **문법**: `{python}from_json( <JSON_문자열>, <설명서> )`
	- **<JSON_문자열>**: `.cast("string")`으로 변환된 컬럼
	- **<설명서>**: `StructType`으로 만든 스키마 변수

```python
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

# 1. 설계도(Schema) 만들기
schema = StructType([
    StructField("name", StringType()),
    StructField("age", IntegerType())
])

# 2. 적용하기 (옵션: map형태로 옵션을 넣을 수 있음)
df.select(
    from_json(col("json_string_col"), schema).alias("data")
).select("data.name", "data.age") # 구조체 안으로 접근 가능
```

### 배열(Array) 형태의 JSON이라면?

데이터가 `[{"name":"A"}, {"name":"B"}]` 처럼 리스트로 들어온다면 스키마를 `ArrayType`으로 감싸야 합니다.

```python
from pyspark.sql.types import ArrayType

# 스키마를 ArrayType으로 감싸기
array_schema = ArrayType(schema)

df.select(from_json(col("json_array_col"), array_schema).alias("data"))
```

---
## `to_json`: 다시 포장하기 (Struct -> String) 

가공이 끝난 데이터를 카프카로 보내거나 파일로 저장하기 위해 **하나의 문자열 덩어리**로 묶는 과정입니다.

- **문법**: `{python}to_json( <묶은_데이터> )`
	- **<묶은_데이터>**: 보통 여러 컬럼을 `struct()`로 묶어서 넣습니다.
### 핵심: `struct()`와 짝꿍

보통 여러 컬럼을 하나로 합쳐야 하므로 `struct()` 함수와 같이 씁니다.

```python
from pyspark.sql.functions import to_json, struct, col

# name과 age 컬럼을 묶어서 JSON 문자열로 변환
df.select(
    to_json(struct(col("name"), col("age"))).alias("value")
)
```

- **결과물:** `{"name": "Gong", "age": 30}` (String 타입)

---
## 스키마 만들기 귀찮을 때 (`get_json_object`) 

`from_json`은 정석이지만 스키마를 짜야 해서 번거롭습니다.
데이터 구조가 복잡하거나, **"딱 하나만 꺼내고 싶을 때"** 는 이 함수를 씁니다.

- **문법**: `{python}get_json_object( <JSON_문자열>, <꺼낼_위치> )`

- **장점:** 스키마 필요 없음.
- **단점:** 한 번에 하나만 꺼낼 수 있음 (느림).

```python
from pyspark.sql.functions import get_json_object

# json_col에서 '$.info.city' 경로에 있는 값만 쏙 빼오기
df.select(
    get_json_object(col("json_col"), "$.info.city").alias("city")
)
```

---
## 요약 비교 (Cheat Sheet)

|**상황**|**추천 함수**|**비유**|
|---|---|---|
|**본격적으로 데이터를 분석해야 할 때**|**`from_json`**|이삿짐을 풀어서 서랍장에 정리하기|
|**다른 시스템(Kafka)으로 전송할 때**|**`to_json`**|이삿짐을 박스에 테이프로 포장하기|
|**급하게 값 하나만 확인하고 싶을 때**|**`get_json_object`**|박스에 구멍 뚫어서 내용물 확인하기|