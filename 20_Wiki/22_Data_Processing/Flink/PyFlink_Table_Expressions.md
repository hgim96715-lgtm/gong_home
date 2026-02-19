---
aliases:
  - PyFlink
  - TAble
  - 조건문
  - cast
  - execute
tags:
  - PyFlink
related:
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_Table_API_Intro]]"
  - "[[PyFlink_SQL_Windows]]"
  - "[[PyFlink_SQL_Integration]]"
---
## 한 줄 요약:

**"SQL을 파이썬 함수로 쪼개놓은 것."** 
(문자열 쿼리(`SELECT * FROM...`) 대신, `select()`, `filter()` 같은 함수로 로직을 짜는 방법)

---
## 표현식(Expressions)이란?

Flink에서 "컬럼"이나 "값"을 다룰 때 사용하는 전용 객체입니다. 
파이썬의 문자열(`"price"`)과 Flink의 컬럼(`col("price")`)은 완전히 다릅니다.

- **문자열 `"price"`:** 그냥 글자.
- **표현식 `col("price")`:** "이 테이블의 'price'라는 컬럼을 찾아가라"는 **지시어**.

### 필수 Import

이 두 줄은 무조건 적고 시작하세요.

```python
from pyflink.table.expressions import col, lit
```

---
## 핵심 함수 2대장: `col` & `lit`

### ① `col(name)`: 컬럼 선택

테이블의 특정 컬럼을 가리킬 때 씁니다.

```python
col("order_id")
col("price")
```

### ② `lit(value)`: 상수(Literal) 입력

데이터가 아닌, 고정된 숫자나 문자를 넣을 때 씁니다.

```python
lit(100)       # 숫자 100
lit("Korea")   # 문자열 "Korea"
```

>**왜 `lit`을 쓰나요?** 
>`col("price") + 10` 처럼 파이썬 숫자를 바로 더해도 되지만, `table.select(col("price"), lit("WON"))` 처럼 **새로운 상수 컬럼을 만들 때**는 필수입니다.

---
## 데이터 조작 (Transformation) 

SQL 문법과 1:1로 매칭됩니다. (직관적임)

### ① SELECT (선택)

원하는 컬럼만 뽑아냅니다.

```python
# SQL: SELECT category, price FROM logs
table.select(col("category"), col("price"))
```

### ② FILTER / WHERE (조건)

원하는 데이터만 남깁니다. (`filter`와 `where`는 똑같습니다.)

```python
# SQL: WHERE price > 5000 AND category = 'Electronics'
table.filter((col("price") > 5000) & (col("category") == "Electronics"))
```

### ③ ALIAS (별칭)

컬럼 이름을 바꿉니다.

```python
# SQL: SELECT count(order_id) AS cnt
col("order_id").count.alias("cnt")
```

### ④ ADD COLUMNS (추가)

기존 컬럼을 이용해 계산된 새 컬럼을 붙입니다.

```python
# 가격(price) * 수량(quantity) = 총액(total) 컬럼 추가
table.add_columns((col("price") * col("quantity")).alias("total"))
```

---
## 테이블 호출과 실행 (Source & Sink)

SQL로 정의한 테이블을 파이썬 객체로 가져오거나(`Source`), 처리된 결과를 화면에 뿌리는(`Action`) 방법입니다.

### ① `from_path("이름")`: 테이블 불러오기

`CREATE TABLE` SQL문으로 등록해둔 테이블을 **Table 객체**로 변환합니다.

- **역할:** SQL 세계(`Catalog`)에 있는 테이블을 파이썬 세계(`Table Object`)로 데려옵니다
- 사용법:

```python
# 1. SQL로 테이블 정의 (DDL)
t_env.execute_sql("""
    CREATE TABLE source_table (
        id STRING,
        price INT
    ) WITH ( ... )
""")

# 2. 파이썬 변수로 변환 (이때부터 .select(), .filter() 사용 가능!)
tab = t_env.from_path("source_table")
```

### ② `execute().print()`: 실행 및 출력

Flink는 **Lazy Evaluation(지연 실행)** 방식입니다. 
코드를 짤 때는 계획(Plan)만 세우고, 이 함수를 호출하는 순간 진짜로 실행됩니다.

- **역할:** 작성된 파이프라인을 실행하고, 결과를 **표준 출력(Console/Log)**에 찍습니다. (디버깅용)
- **구조:** `.execute()` (실행해라) + `.print()` (화면에 보여줘라)
- 사용법:

```python
# 변수에 담긴 로직을 즉시 실행
result_table.execute().print()
```

>**주의:** 도커 환경에서는 터미널이 아니라 **TaskManager 로그(`docker logs ...`)** 에 출력됩니다.


---
## 집계 함수 (Aggregation) 

데이터를 묶어서 계산할 때 사용합니다. 
주로 `.group_by()` 뒤에 옵니다.

|**함수**|**설명**|**예시 코드**|
|---|---|---|
|**`count`**|개수 세기|`col("order_id").count`|
|**`sum`**|합계|`col("price").sum`|
|**`avg`**|평균|`col("price").avg`|
|**`max` / `min`**|최대/최소|`col("price").max`|
|**`distinct`**|중복 제거|`col("user_id").distinct()`|

---
## 실전 예제: 카테고리별 매출 집계

**목표:**

1. `source_table`에서
2. 가격(`price`)이 0보다 큰 것만 남기고
3. 카테고리(`category`)별로 묶어서
4. 매출 합계(`total_sales`)를 구한다.

```python
# 1. 원본 테이블
tab = t_env.from_path("source_table")

# 2. 로직 적용 (Chaining 방식)
result_tab = tab \
    .filter(col("price") > 0) \
    .group_by(col("category")) \
    .select(
        col("category"),
        col("price").sum.alias("total_sales"),
        col("order_id").count.alias("order_count")
    )

# 3. 결과 출력
result_tab.execute().print()
```

---

## 주의사항 (Common Pitfalls) ⚠️

1. **파이썬 내장 함수 쓰지 마세요:**
    
    - ❌ `sum(col("price"))` (파이썬 내장 sum 사용 불가)
    - ✅ `col("price").sum` (Flink 표현식 사용)
        
2. **논리 연산자:**
    
    - `and`, `or` 대신 **`&`, `|`** 기호를 써야 합니다.
    - 조건문은 괄호 `()`로 감싸는 게 안전합니다.
    - 예: `(col("a") > 1) & (col("b") < 5)`
        
3. **타입 호환성:**

    - 문자열 컬럼에 숫자를 더하려고 하면 에러가 납니다. (`cast` 필요)