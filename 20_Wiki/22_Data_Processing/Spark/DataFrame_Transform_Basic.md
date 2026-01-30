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




