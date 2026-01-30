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
  - "[[DataFrame_Aggregation]]"
  - "[[DataFrame_Transform_Basic]]"
  - "[[00_Apache_Spark_HomePage]]"
linked: file:///Users/gong/gong_study_de/apache-spark/notebooks/step11.ipynb
---
##  개요 

`pyspark.sql.functions` 모듈의 필수 함수를 정리한 **마스터 사전**입니다.

* 모든 예제는 `import pyspark.sql.functions as F` 가 되어있다고 가정합니다.
* **Ctrl + F** 로 필요한 기능을 검색해서 사용하세요.

---
## 기본 & 상수 (Basic) 

가장 기초가 되는 함수들입니다.

### `F.col("name")`

* **역할:** 컬럼을 선택하는 가장 안전한 방법.
* **꿀팁:** 문자열(`"age"`) 대신 써야 연산(`+`), 별명(`.alias`), 정렬(`.desc`)이 가능합니다.

```python
df.select(F.col("age") + 1)
```


### `F.lit(value)`

- **역할:** **상수(Literal)** 값을 컬럼 객체로 만듭니다.
- **사용처:**
    1. 모든 행에 똑같은 값을 넣을 때
    2. 연산할 때 숫자/문자를 끼워 넣을 때
    3. **빈 컬럼(Null)** 을 억지로 만들 때 (중요!)

```python
# 모든 행에 "Korea" 넣고, 빈 컬럼(null_col) 만들기
df.withColumn("country", F.lit("Korea"))\
  .withColumn("null_col", F.lit(None)).show()
```

> ** Q. 그냥 `None` 넣으면 안 되나요?**
> **A. 절대 안 됩니다!** 스파크는 `Column` 객체만 취급하기 때문에, 파이썬의 `None`을 던져주면 "이게 뭐임?" 하고 에러를 뱉거나 무시합니다.
>
> **[실전 상황]** 두 테이블을 합칠 때(Union), 컬럼 개수를 맞추기 위해 억지로 빈 컬럼을 만들어야 할 때가 있습니다.
> 
```python
# [X] 에러: Schema 추론 불가
df.withColumn("empty_col", None)

# [O] 정답: 빈 컬럼 생성 성공!
df.withColumn("empty_col", F.lit(None)) > ```
```



---
## 조건문 (Logic) 뇌지컬 

엑셀의 IF, SQL의 CASE WHEN과 같습니다.

### `F.when().otherwise()`

- **역할:** 조건에 따라 다른 값을 넣습니다.
- **구조:** `{python}.when(조건, 참일때_값).otherwise(거짓일때_값)`

```python
df.withColumn("status", 
    F.when(F.col("age") < 20, "미성년")
     .when(F.col("age") < 65, "성인")
     .otherwise("노인")
).show()
```

### `F.coalesce(col1, col2, ...)` 

- **역할:** 여러 컬럼 중 **Null이 아닌 첫 번째 값**을 가져옵니다.
- **사용처:** 데이터 빵꾸(Null) 때울 때 필수!

```python
# phone1이 없으면 phone2를, 그것도 없으면 "No Phone"을 넣어라
df.select(F.coalesce(F.col("phone1"), F.col("phone2"), F.lit("No Phone")))
```

---
## 문자열 처리 (String) 

텍스트 데이터를 지지고 볶을 때 씁니다.

### `F.substring(col, pos, len)`

- **역할:** 글자를 자릅니다.
- **주의:** **시작 인덱스가 0이 아니라 1입니다!** (SQL 표준)

```python
# 20240101 -> 2024 (1번째부터 4글자)
df.select(F.substring("date_str", 1, 4))
```

### `F.split(col, pattern)`

- **역할:** 문자를 쪼개서 **리스트(Array)** 로 만듭니다.

```python
# "apple,banana" -> ["apple", "banana"]
df.select(F.split("fruits", ","))
```


### `F.concat(col1, col2, ...)`

- **역할:** 여러 문자를 하나로 합칩니다.

```python
# "Seoul", "Korea" -> "Seoul_Korea"
df.select(F.concat(F.col("city"), F.lit("_"), F.col("country")))
```

### `F.regexp_replace(col, pattern, replacement)` 

- **역할:** 정규표현식으로 문자를 찾아 바꿉니다. (가장 강력한 바꾸기 도구)

```python
# 전화번호의 '-'를 공백('')으로 제거
df.withColumn("clean_phone", F.regexp_replace("phone", "-", ""))
```

---
## 날짜 & 시간 (Date & Time) 

### `F.current_date()`

- **역할:** 오늘 날짜(YYYY-MM-DD)를 가져옵니다.

### `F.to_date(col, format)`

- **역할:** 문자열을 진짜 날짜 타입으로 바꿉니다.

```python
# "20241225" -> 2024-12-25 (DateType)
df.select(F.to_date("date_str", "yyyyMMdd"))
```

### `F.date_add(col, days)` / `F.date_sub()`

- **역할:** 날짜 더하기 / 빼기

```python
# 오늘로부터 7일 뒤
df.select(F.date_add(F.current_date(), 7))
```

### `F.datediff(end, start)`

- **역할:** 두 날짜 사이의 **일수(Day) 차이**를 구합니다.

```python
# 퇴사일 - 입사일 = 근속일수
df.select(F.datediff(F.col("end_date"), F.col("start_date")))
```

---
## 자 & 통계 (Number) 

### `F.round(col, scale)`

- **역할:** 반올림합니다. 
- `scale`은 소수점 자릿수입니다.

### `F.rand()`

- **역할:** 0~1 사이의 랜덤 실수를 만듭니다. 
- 샘플링할 때 씁니다.

### `F.avg()`, `F.sum()`, `F.max()`, `F.min()`

- **역할:** 집계 함수들. 주로 `groupBy`와 짝꿍입니다.

### `F.countDistinct(col)`

- **역할:** **중복을 제거하고** 개수를 셉니다. (유니크 유저 수 셀 때 필수)

```python
df.agg(F.countDistinct("user_id"))
```

---
## 배열 & 구조체 (Collection) 

스파크의 강력한 기능인 배열(Array) 처리입니다.

### `F.size(col)`

- **역할:** 배열의 크기(길이)를 반환합니다.

```python
# ["A", "B"] -> 2
df.select(F.size(F.col("my_array")))
```

### `F.explode(col)` 

- **역할:** 배열을 행(Row)으로 **폭파(Explode)** 시킵니다.
- **상황:** `split`으로 쪼갠 뒤에, 리스트 요소를 각각 별도 행으로 만들 때 씁니다.

```python
# [Row(items=["A", "B"])]
#  ↓ explode ↓
# [Row(col="A"), Row(col="B")]
df.select(F.explode(F.split("text", " ")))
```

