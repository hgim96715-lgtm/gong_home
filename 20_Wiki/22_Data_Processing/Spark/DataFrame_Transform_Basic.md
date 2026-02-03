---
aliases:
  - DataFrame변환
  - Select
  - Filter
  - withColumn
  - 전처리
tags:
  - Spark
  - Transformation
related:
  - "[[DataFrame_Aggregation]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_DataFrame_Basics]]"
  - "[[Spark_Functions_Library]]"
  - "[[Spark_Data_Cleaning]]"
  - "[[Spark_Streaming_JSON_ETL_Project]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step9.ipynb
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step7.ipynb
---
## 개념 한 줄 요약 

**"엑셀에서 열 숨기고(Select), 필터 걸고(Filter), 피벗테이블 돌리는(GroupBy) 작업을 코드로 하는 것."**

* **핵심 도구:** 스파크의 모든 변환 작업은 **`pyspark.sql.functions`** 모듈에 있는 도구들을 가져와서 씁니다.
- **대원칙 (불변성):** 모든 조작(`select`, `filter` 등)은 **원본을 절대 바꾸지 않고, 새로운 데이터프레임을 만듭니다.**
	- **변경된 결과를 쓰려면 반드시 변수에 다시 담으세요!** (`df = df.select(...)`)
	- 🔗 참고: [[Spark_DataFrame_Basics#[핵심] 불변성 (Immutability)|불변성 자세히 보기]]


---
## 임포트 국룰 (Import Strategy) 

함수가 너무 많아서(`col`, `sum`, `avg`, `lit`...) 하나씩 임포트하면 코드가 지저분해집니다.
현업에서는 보통 **`F`** 라는 별명으로 통째로 가져와서 씁니다.

```python
# [하수] 하나하나 다 가져오기 (비추천 )
from pyspark.sql.functions import col, asc, desc, sum, avg, max

# [고수] 모듈 통째로 가져오기 (추천 )
import pyspark.sql.functions as F

# 사용법: F.col("age"), F.sum("salary") 처럼 앞에 F를 붙임.
```

---
## 컬럼 객체 (Column Object) 완벽 정복

### ① 컬럼을 부르는 3가지 방법

```python
# 1. [강추] F.col 사용 (가장 안전하고 강력함) 
df.select(F.col("age"))

# 2. DataFrame 속성 사용 (간편하지만 공백/특수문자 있으면 에러남)
df.select(df.age)

# 3. 딕셔너리 방식 (Pandas 스타일, 컬럼명에 공백 있어도 됨)
df.select(df["age"])
```

### ② 왜 `col` 객체를 써야 하나요?

문자열(`"age"`)은 그냥 이름일 뿐이지만, `col` 객체는 **연산이 가능한 상태**입니다.

- **사칙연산:** `"age" + 1` (에러 ) vs `F.col("age") + 1` (성공 )
- **조건비교:** `"age" > 20` (문자열 비교됨) vs `F.col("age") > 20` (필터 조건됨)
- **메서드 체이닝:** `.alias()`, `.cast()`, `.desc()` 등은 **객체 뒤에만** 붙일 수 있습니다.

### ③ [주의] 컬럼 이름 자동 변경 규칙 (Auto-Naming)

스파크는 **연산(계산)을 하는 순간, 컬럼 이름을 수식 그대로 바꿔버립니다.**

```python
df.select(F.col("age") + 1).show()
# 결과 컬럼명: "(age + 1)"  <-- 괄호랑 더하기 기호가 이름에 다 들어감! 
```

👉 **해결책:** 연산을 했다면, **반드시** 뒤에 `.alias("이름")`를 붙여서 깔끔하게 정리해줘야 합니다.


----
## 선택하기 (`select`) & 별명 짓기 (`alias`)

원하는 열만 쏙 뽑아내거나, 이름을 바꿔서 가져옵니다.

### ① 기본 선택

```python
# 1. 문자열로 선택 (가장 단순)
df.select("name", "age").show()

# 2. col 객체로 선택 (기능을 더 쓸 수 있음)
df.select(F.col("name"), F.col("age")).show()
```

### ② 별명 붙이기 (`alias`) 

**주의:** 문자열(`"name"`)에는 `.alias`를 못 붙입니다. 
반드시 **`F.col()`** 로 감싸야 합니다!

```python
# [O] 올바른 방법
df.select(
    F.col("name"),
    (F.col("age") + 1).alias("korean_age")  # 나이에 1 더해서 가져오기
).show()
```

---
## 컬럼 관리 (추가 및 이름 변경) 

`select`가 조회를 한다면, 이 기능들은 데이터를 **추가하거나 수정**할 때 씁니다. 
이름이 비슷해서 헷갈리기 쉬우니 주의하세요!

### ① 컬럼 추가/수정하기 (`withColumn`) 

내용물(데이터)을 **새로 만들거나 고치는** 작업입니다. (비용 발생), 컬럼객체가 들어가야함
- **문법:** `{python}.withColumn("새로_만들_이름", 계산_로직_객체)`
- **주의:** 두 번째 칸에 그냥 숫자(`10`)나 문자(`"Korea"`)를 넣으면 에러 납니다.  
- **`F.lit()`** 이나 **`F.col()`** 로 감싸야 합니다.

```python
# 1. 'country' 컬럼을 만들고 전부 'Korea'로 채우기 (상수)
df.withColumn("country", F.lit("Korea")).show()

# 2. 'age' 컬럼에 1을 더해서 'korean_age' 만들기 (계산)
df.withColumn("korean_age", F.col("age") + 1).show()

# 3. [주의] 기존 컬럼명과 똑같이 쓰면? -> 덮어쓰기(Overwrite) 됨!
df.withColumn("age", F.col("age") + 1).show()
```

### ② 컬럼 이름만 바꾸기 (`withColumnRenamed`) 

데이터는 그대로 두고 **이름표(Header)** 만 바꿔 답니다. , 문자열만 들어감  (매우 빠름)

- **문법:** `{python}.withColumnRenamed("원래_이름", "새_이름")`
- **주의:** 없는 컬럼 이름을 적어도 에러는 안 나고, 그냥 아무 일도 안 일어납니다. (무시됨)

```python
# "age"를 "user_age"로 이름 변경
df.withColumnRenamed("age", "user_age").show()
```

---

## 걸러내기 (`filter` / `where`)

원하는 행(Row)만 남깁니다. `filter`와 `where`는 **완벽하게 똑같은 함수**입니다.

### ① 두 가지 스타일

```python
# [방법 1] SQL 스타일 (문자열) - SQL 문법 그대로 씀! 
# 장점: 짧고 직관적임. 
# 단점: 오타 나면 실행해봐야 앎.
df.filter("age > 30").show()
df.filter("dept = 'IT'").show()  # 문자열 비교는 작은따옴표('') 주의

# [방법 2] 객체 스타일 (F.col) - 파이썬스러움 
# 장점: 자동완성 지원, 복잡한 로직 구현 가능.
# 단점: 코드가 좀 길어짐.
df.filter(F.col("age") > 30).show()
df.filter(F.col("dept") == "IT").show()
```

### ② 조건 여러 개 (AND `&`, OR `|`) 

**매우 중요:** 객체 스타일 사용 시 파이썬 연산자 우선순위 때문에 **반드시 괄호 `( )`로 조건을 감싸야 합니다!**

```python
# [O] 30살 넘고(AND) 부서가 IT인 사람
df.filter( (F.col("age") > 30) & (F.col("dept") == "IT") ).show()

# [꿀팁] SQL 스타일은 괄호 없어도 됨 (AND, OR 문자로 씀)
df.filter("age > 30 AND dept = 'IT'").show()
```

### ③ [핵심] 결측치(Null) 처리 (`isNotNull`,`isNull`) 

`{text}== None`은 절대 동작하지 않습니다.

```python
# [X] 절대 안 됨! (에러는 안 나는데 결과가 이상함)
df.filter(F.col("email") == None) 

# [O] 올바른 방법 (객체 스타일)
df.filter(F.col("email").isNull()).show()    # Null인 것만 보기
df.filter(F.col("email").isNotNull()).show() # Null이 아닌 것만 보기 (Clean Data)

# [O] 올바른 방법 (SQL 스타일)
df.filter("email IS NULL").show()
df.filter("email IS NOT NULL").show()
```

> 결측치 처리에 대해 더 보고싶다면 [[Spark_Data_Cleaning#결측치 현황 파악하기 (Null Check)|filter_null]] 참조

---
## 데이터 타입 변경 (`.cast`) 

데이터의 **자료형(Type)** 을 바꿀 때 사용합니다. 숫자가 문자로 저장되어 있거나, 날짜를 문자로 바꾸고 싶을 때 필수입니다.

- **문법:** `{python}col("컬럼명").cast("바꿀타입")` 
- **주요 타입:** `"string"`, `"integer"`, `"double"`, `"date"`, `"timestamp"`

### 사용 예시

**상황:** 카프카에서 온 `value` 컬럼은 0과 1로 된 **이진 데이터(Binary)** 상태입니다. (사람이 못 읽음)
**해결:** "이거 문자열(String)이니까 글자로 보여줘!"라고 **강제로 변환**한 것입니다. 그래야 다음 단계인 `from_json`이 읽을 수 있거든요.

```python
from pyspark.sql.functions import col

# 1. [핵심] Binary -> String 변환 (카프카 필수 패턴)
# 암호 같은 기계어를 사람이 읽을 수 있는 글자로 변경
df.select(F.col("value").cast("string"))


# 2. 일반적인 숫자 변환 예시
# 문자로 된 "100"을 계산 가능한 숫자 100으로 변경
df = df.withColumn("age", F.col("age").cast("integer"))

# 3. 여러 컬럼 한 번에 바꾸기
df = df.selectExpr( 
	"cast(value as string) as json_string",
	"cast(age as int) as age_int" 
)
```

---
##  SQL 문법 바로 쓰기 (`selectExpr`) 

`select`를 쓰다 보면 `F.col()` 치고, `.alias()` 치고... 괄호 닫느라 손가락이 아픕니다.
`selectExpr` (Select Expression)은 **"Python 코드 안에서 SQL 문법을 그대로 쓰게 해주는 마법의 함수"** 입니다.

* **특징:** 따옴표(`" "`) 안에 SQL 쿼리 짜듯이 적으면 알아서 해석해줍니다.
* **장점:** 코드가 훨씬 짧고 직관적입니다.

###  비교 체험 (일반 select vs selectExpr)

**미션:** "나이(age)에 1을 더하고, 이름을 'user_name'으로 바꾸고, 타입을 숫자로 바꿔라."

```python
# [방법 1] 일반 select (F.col 지옥 )
from pyspark.sql.functions import col

df.select(
    (col("age") + 1).alias("next_year_age"),  # 1. 더하기 + 별명
    col("name").alias("user_name"),           # 2. 이름 변경
    col("salary").cast("int")                 # 3. 타입 변경
).show()

# [방법 2] selectExpr (SQL 천국 ✨) - 훨씬 깔끔함!
df.selectExpr(
    "age + 1 as next_year_age",      # 1. SQL 문법 그대로!
    "name as user_name",             # 2. AS로 별명 짓기
    "cast(salary as int) as salary"  # 3. cast 함수도 SQL처럼 사용
).show()
```

### 언제 쓰면 좋나요?

1. **계산식이 복잡할 때:** `{python}selectExpr("price * 0.9 + 100 as final_price")`
2. **타입 변환(Cast)할 때:** `{python}selectExpr("cast(value as string)")`
3. **CASE WHEN 문 쓸 때:** 파이썬 `{python}when().otherwise()`보다 SQL이 익숙하다면 강추!

---
## 컬럼 안에서 SQL 수식 쓰기 (`expr`) 

`withColumn`이나 `filter`를 쓸 때, 파이썬 문법보다 SQL 문법이 더 편할 때가 있습니다. 이때 `expr`을 사용합니다.

- **문법**: `{python}expr( "SQL_수식" )`
- **비유**: 파이썬 코드 속에 넣는 **"작은 SQL 조각"**

### ✅ 실전 예시

```python
from pyspark.sql.functions import expr

# 1. 계산식 활용 (괄호 지옥 탈출)
# 일반: (F.col("price") * 0.9).alias("discount")
df.withColumn("discount", expr("price * 0.9"))

# 2. 조건문(CASE WHEN) 활용 (가장 많이 쓰임!)
df.withColumn("age_group", expr("CASE WHEN age >= 20 THEN 'Adult' ELSE 'Minor' END"))

# 3. 사용자님 코드에서의 활용
# 'behavior'라는 별명을 SQL 수식으로 처리
df.withColumn("behavior", expr("behavior"))
```

- `selectExpr`: **통째로** SQL로 조회하고 싶을 때
- `expr`: **특정 컬럼** 하나만 SQL 수식으로 계산하고 싶을 때

---
## 여러 컬럼 하나로 묶기 (`struct`) 

여러 개의 컬럼을 **하나의 구조체(Struct, 바구니)** 로 묶을 때 사용합니다.
주로 데이터를 계층적(Hierarchy)으로 만들거나, **`to_json`을 쓰기 전 단계**에서 필수적으로 사용됩니다.

* **개념:** 흩어져 있는 컬럼들을 하나의 세트로 포장함.
* **문법:** `{python}struct( col("컬럼1"), col("컬럼2"), ... )`

### 사용 예시

**상황:** `name`과 `age` 컬럼이 따로 있는데, 이걸 `info`라는 하나의 컬럼 안에 넣고 싶음.

```python
from pyspark.sql.functions import struct, col

# 1. 'info'라는 새 컬럼을 만들고, 그 안에 name과 age를 묶어서 넣음
df.withColumn("info", struct(col("name"), col("age"))).show(truncate=False)
```

**결과 비교**
- **Before:** `| name | age |` (컬럼 2개)
- **After:** `| info |` (컬럼 1개, 내용은 `{"name": "Gong", "age": 30}`)

### 💡 [실전 꿀팁] 왜 묶나요?

1. **JSON 만들기 전:** `to_json` 함수는 **컬럼 1개**만 입력으로 받습니다. 그래서 여러 데이터를 JSON으로 만들려면 먼저 `struct`로 묶어서 하나로 만들어줘야 합니다.
2. **데이터 정리:** 주소 관련 컬럼이 5개(`시`, `구`, `동`, `번지`, `우편번호`)라면, `address`라는 struct 하나로 묶어서 관리하면 깔끔합니다.
