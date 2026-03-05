---
aliases:
  - SELECT
  - FROM
  - SQL 산술연산
  - SQL 합성연산
  - 별칭
  - AS
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DML_CRUD]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[SQL_Understanding_NULL]]"
  - "[[SQL_DISTINCT_vs_GROUP_BY]]"
---
# SQL SELECT FROM — 데이터 조회의 시작점

## 개념 한 줄 요약

> **"어떤 테이블의 어느 컬럼을 가져올지 지정하는 가장 기초적인 명령어이자, 가져오면서 동시에 가공(계산·문자열 합성)까지 해주는 만능 도구."**

---

## 왜 필요한가?

데이터베이스에는 수백 개의 테이블, 테이블마다 수백 개의 컬럼이 있을 수 있다. `SELECT *` (전체 조회) 는 수억 건의 데이터를 한 번에 뿌리려다 **서버가 멈추거나(OOM), 클라우드 비용 폭탄** 을 맞게 된다.

> **BigQuery 같은 클라우드 환경에서 `SELECT *` 는 '해고 사유' 라는 농담이 있을 정도다.** 1TB 테이블에서 `*` → 1TB 비용 / 컬럼 하나만 지정 → 10MB 비용으로 끝날 수 있다. 데이터 엔지니어는 항상 **"최소한의 컬럼만 가져오고 있는가?"** 를 고민해야 한다.

---

## 기본 구조

```sql
SELECT
    product_name AS name,          -- 1. 단순 조회 + 별칭
    price * 0.9  AS discounted_price,  -- 2. 산술 연산
    category || '_' || product_name AS full_code  -- 3. 문자열 합성 (Oracle)
FROM
    coupang_products;              -- 4. 어느 테이블에서?
```

---

---

## ① 연산자 우선순위

SQL 도 수학처럼 연산 순서가 정해져 있다. 괄호가 최우선, OR 이 꼴찌.

| 순위  | 종류             | 기호·키워드                                                   | 암기 포인트                             |
| :-: | -------------- | -------------------------------------------------------- | ---------------------------------- |
|  1  | 괄호             | `( )`                                                    | 우선순위를 강제로 끌어올리고 싶으면 무조건 괄호         |
|  2  | 산술 (곱·나눗셈·나머지) | `*` `/` `%`                                              | `%`(나머지) 는 곱하기·나누기와 동급             |
|  3  | 산술 (더하기·빼기)    | `+` `-`                                                  |                                    |
|  4  | 연결             | <code>\|\|</code> (Oracle·PostgreSQL) / `+` (SQL Server) | 문자열을 이어 붙이는 연산자. 산술 연산 후, 비교 전에 실행 |
|  5  | 비교             | `{text}=` `>` `<` `>=` `<=` `<>`                         | 양쪽 값을 비교                           |
|  6  | SQL 전용         | `BETWEEN` `IN` `LIKE` `IS NULL`                          | SQL 에서만 쓰는 특수 필터                   |
|  7  | 논리 (부정)        | `NOT`                                                    | 결과를 뒤집음                            |
|  8  | 논리             | `AND`                                                    | 🚨 AND 가 OR 보다 먼저! 헷갈리는 함정 포인트     |
|  9  | 논리             | `OR`                                                     | 가장 마지막에 계산                         |

> 🚨 **NULL 주의:** 컬럼끼리 산술·연결 연산 시 식에 `NULL` 이 단 하나라도 있으면 결과는 무조건 `NULL`.

```sql
-- 연결 연산자 예시
SELECT '홍' || '길동'       -- 결과: '홍길동'   (Oracle·PostgreSQL)
SELECT '홍' + '길동'        -- 결과: '홍길동'   (SQL Server)
SELECT '이름: ' || NULL     -- 결과: NULL       (NULL 전파!)
```
---

# ② 산술 연산자

숫자형 컬럼은 `SELECT` 절과 `WHERE` 절에서 바로 사칙연산이 가능하다.

```sql
-- 원가에서 10% 할인된 가격을 즉석에서 계산
SELECT
    product_name,
    price,
    price * 0.9   AS discounted_price,  -- 10% 할인가
    price * 0.9 * 1.1 AS vat_price      -- 할인가에 부가세 10% 추가
FROM products;

-- WHERE 절에서도 계산 가능
SELECT * FROM products
WHERE price * 0.9 < 10000;  -- 할인가가 1만원 미만인 상품만
```

---

---

# ③ 합성 연산자 — 문자열 이어 붙이기

두 개 이상의 문자열·컬럼을 **하나로 이어 붙일 때** 사용한다. DBMS 마다 쓰는 기호가 다르다.

| DB                  | 연산자·함수                      | 예시                                              |
| ------------------- | --------------------------- | ----------------------------------------------- |
| Oracle / PostgreSQL | <code>\|\|</code> (수직선 두 개) | <code>'홍' \|\| '길동'</code> → <code>'홍길동'</code> |
| SQL Server (MSSQL)  | `+`                         | `'홍' + '길동'` → `'홍길동'`                          |
| MySQL / 표준 SQL      | `CONCAT()`                  | `CONCAT('홍', '길동')` → `'홍길동'`                   |

```sql
-- Oracle / PostgreSQL: || 로 이어 붙이기
SELECT
    category || '_' || product_name AS full_code
FROM products;
-- 결과: '전자제품_아이폰15'

-- 구분자와 함께 여러 컬럼 합치기
SELECT
    last_name || ' ' || first_name AS full_name
FROM employees;
-- 결과: '김 철수'

-- MySQL: CONCAT 함수
SELECT CONCAT(category, '_', product_name) AS full_code FROM products;
```

> **🚨 NULL 주의:** `||` 연산에서도 어느 한 쪽이 `NULL` 이면 결과가 `NULL` 이 된다. Oracle 에서는 `NVL(컬럼, '')`, PostgreSQL 에서는 `COALESCE(컬럼, '')` 로 NULL 을 빈 문자열로 치환하고 나서 합성하자.

```sql
-- NULL 안전하게 합성하기 (PostgreSQL)
SELECT COALESCE(first_name, '') || ' ' || COALESCE(last_name, '') AS full_name
FROM employees;
```

---

---

# ④ 별칭 (Alias)

연산 후 컬럼명이 `price * 0.9` 처럼 수식 그대로 나와 지저분해진다. `AS` 키워드로 예쁜 이름표를 달아준다.

sql

```sql
SELECT
    price * 0.9 AS discounted_price,  -- AS 사용 (권장)
    price * 0.9    discounted_price    -- AS 생략 가능 (비권장)
FROM products;
```

---

## 별칭 미지정 시 컬럼명 — DB 별 동작 차이 ⭐️

> **Oracle 은 별칭을 지정하지 않으면 컬럼명이 대문자로 출력된다.**

```sql
-- Oracle: 별칭 없이 조회
SELECT price * 0.9 FROM products;
-- 결과 컬럼명: PRICE*0.9  ← 자동으로 대문자 + 수식 그대로 노출

-- Oracle: 별칭 지정 (소문자로 쓰면?)
SELECT price * 0.9 AS discounted_price FROM products;
-- 결과 컬럼명: DISCOUNTED_PRICE  ← 쌍따옴표 없으면 대문자로 변환!

-- Oracle: 소문자 별칭을 유지하려면 쌍따옴표 필수
SELECT price * 0.9 AS "discounted_price" FROM products;
-- 결과 컬럼명: discounted_price  ← 쌍따옴표로 감싸야 소문자 유지
```

|DB|별칭 미지정 시 컬럼명|별칭 쌍따옴표 없을 때|
|---|---|---|
|**Oracle**|**대문자** 로 표시|별칭도 **대문자** 로 변환|
|**PostgreSQL**|소문자 로 표시|별칭도 소문자 로 변환|
|**SQL Server**|수식 또는 열 이름 그대로|그대로 유지|

> SQLD 시험에서 Oracle 기준으로 출력 결과를 묻는 문제가 나올 때, 별칭 없이 쓴 컬럼이 **대문자** 로 나온다는 점을 반드시 기억해야 한다.

---

## 쌍따옴표가 필요한 경우 ⚠️

|상황|예시|이유|
|---|---|---|
|별칭에 공백 포함|`AS "할인 가격"`|공백 있으면 반드시 쌍따옴표|
|특수문자 포함|`AS "price(원)"`|특수문자 포함 시 쌍따옴표|
|대소문자 엄격 구분|`AS "ProductName"`|쌍따옴표 없으면 DB 가 소문자(또는 대문자)로 변환|

```sql
-- ✅ 올바른 사용
SELECT price * 0.9 AS "할인 가격" FROM products;

-- ❌ 절대 안 됨: 홑따옴표는 '값(데이터)' 을 의미
SELECT price * 0.9 AS '할인 가격' FROM products;
--                     ↑
--                     홑따옴표 = 문자열 데이터 (값)
--                     쌍따옴표 = 컬럼·객체 이름
```

---

---

# ⑤ DISTINCT — 중복 제거

`SELECT` 절에서 중복된 행을 제거할 때 사용한다.

```sql
-- 배송이 갔던 도시 목록만 중복 없이 보기
SELECT DISTINCT city FROM orders;

-- 여러 컬럼: (city + district) 세트가 모두 동일한 행만 중복 제거
SELECT DISTINCT city, district FROM orders;
```

> **⚠️ 핵심:** `DISTINCT` 뒤에 컬럼이 여러 개면 **나열된 컬럼 값이 모두 동일한 행** 만 중복으로 처리한다. `(서울, 강남)` 과 `(서울, 마포)` 는 city 가 같아도 district 가 다르면 **둘 다 출력** 된다.

```sql
-- ❌ 절대 불가: 특정 컬럼 하나에만 DISTINCT 씌우기
SELECT col1, DISTINCT(col2) FROM table;

-- ✅ 올바른 위치: SELECT 바로 뒤에 한 번만
SELECT DISTINCT col1, col2 FROM table;
```

> DISTINCT vs GROUP BY · COUNT(DISTINCT) 상세 비교 → [[SQL_DISTINCT_vs_GROUP_BY]] 참고

---

---

# 초보자가 자주 착각하는 포인트

## SQL 은 대소문자 구별을 하나?

- **SQL 문법(명령어):** 대소문자 무관. `select` 나 `SELECT` 나 똑같이 작동.
- **관행:** 명령어는 대문자(`SELECT`), 테이블·컬럼명은 소문자(`user_id`) 로 쓰는 것이 데이터 엔지니어들의 국룰.

## 컬럼 순서가 결과에 영향이 있나?

- 데이터 자체에는 영향 없음. 하지만 **결과표에서 컬럼이 출력되는 순서** 가 바뀐다.
- 보고 싶은 순서대로 적으면 된다.

## `AS '할인 가격'` (홑따옴표) 하면 안 되나?

```
홑따옴표 ' '  =  문자열 데이터 (값)
쌍따옴표 " "  =  컬럼명·테이블명 (객체 이름)
```

별칭은 컬럼의 이름표(객체 이름)이므로 반드시 **쌍따옴표** 를 써야 한다. SQLD 핵심 오답 함정 포인트.

---
