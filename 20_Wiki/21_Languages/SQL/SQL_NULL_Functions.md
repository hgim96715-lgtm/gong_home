---
aliases:
  - NULL 함수
  - NVL
  - COALESCE
  - ISNULL
  - NULLIF
  - NVL2
tags:
  - SQL
related:
  - "[[SQL_Understanding_NULL]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_Type_Casting]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Filtering_WHERE]]"
---


# SQL NULL 관련 함수

## 개념 한 줄 요약

> **"연산을 집어삼키는 블랙홀(NULL) 을 안전한 값으로 메워주는 심폐소생술."**

## 왜 필요한가?

```
① 블랙홀 연산 방지:  100 + NULL = NULL
   보너스가 NULL 인 직원의 총 포인트가 통째로 NULL 이 되는 대참사 방지

② 통계 왜곡 방지:   AVG() 는 NULL 을 분모에서 제외
   0점 처리해야 할 유저가 빠지면 평균이 비정상적으로 높아짐

③ UI/UX 개선:       화면에 'null' 글자 노출 방지
   → '미입력', '0원' 으로 예쁘게 변환
```

---

---

# ① COALESCE — 만국 공통어 (표준 SQL)

> **나열된 값 중 가장 처음으로 NULL 이 아닌 값을 반환한다.** Oracle · PostgreSQL · SQL Server 모두 지원.

```
문법: COALESCE(값1, 값2, 값3, ... , 최후의_보루)
```

```sql
SELECT COALESCE(NULL, NULL, 'C', 'D');   -- 결과: 'C'  (처음으로 NULL 아닌 값)
SELECT COALESCE('A',  NULL, 'C');        -- 결과: 'A'  (첫 번째가 NULL 아님)
SELECT COALESCE(NULL, NULL);             -- 결과: NULL  ← 모두 NULL 이면 NULL 반환
```

> **모든 인자가 NULL 이면 결과도 NULL 이다.** 마지막 인자를 항상 고정 대체값으로 써야 안전하다.

```sql
-- 실무 활용: 핸드폰 없으면 집전화, 그것도 없으면 '연락처 없음'
SELECT COALESCE(phone_num, home_num, '연락처 없음') FROM users;
--                                   ↑ 마지막에 고정 대체값 = NULL 방어막
```

---

---

# ② NVL / ISNULL — 2개만 콕 집어 교체

> **딱 2개의 인자만 받는다.** `(검사할_컬럼, NULL 일 때 대체값)`

```
NULL 아니면  →  원래 값 그대로 반환
NULL 이면    →  두 번째 인자(대체값) 반환
```

## DB 별 함수명

|DB|함수|비고|
|---|---|---|
|**Oracle**|`NVL()`|Null VaLue 의 약자|
|**SQL Server**|`ISNULL()`||
|**PostgreSQL**|없음|`COALESCE()` 사용|

```sql
-- Oracle
SELECT NVL(bonus, 0)      FROM salary;   -- bonus 가 NULL 이면 0
SELECT NVL(name, '무명')  FROM users;    -- name 이 NULL 이면 '무명'

-- SQL Server
SELECT ISNULL(bonus, 0)   FROM salary;
SELECT ISNULL(name, '무명') FROM users;
```

---

---

# ③ NULLIF — 같으면 NULL 로 폭파

> **두 값이 같으면 NULL, 다르면 첫 번째 값을 반환한다.** 0으로 나누기(Divide by Zero) 에러 방지에 최적.

```
문법: NULLIF(값1, 값2)

값1 = 값2  →  NULL 반환
값1 ≠ 값2  →  값1 반환
```

```sql
SELECT NULLIF('A', 'A');   -- 결과: NULL  (같으니까 폭파)
SELECT NULLIF('A', 'B');   -- 결과: 'A'   (다르니까 첫 번째 값 반환)

-- 실무: 분모가 0 일 때 에러 방지
SELECT amount / NULLIF(total_count, 0) FROM sales;
-- total_count = 0 이면 → NULLIF 가 NULL 로 변환
-- → 분모가 NULL 이면 에러 없이 결과도 NULL 로 처리됨
```

---

---

# ④ NVL2 — Oracle 전용 3단 콤보

> **NULL 여부에 따라 서로 다른 두 값을 반환하는 Oracle 전용 함수.** IF-ELSE 로직과 동일.

```
문법: NVL2(검사할_컬럼, NULL이_아닐_때_값, NULL일_때_값)
```

```sql
-- 보너스가 있으면 '대상자', NULL 이면 '제외'
SELECT NVL2(bonus, '대상자', '제외') FROM employees;
```

## ⚠️ NVL2 는 인자가 반드시 3개여야 한다

```sql
-- ❌ 에러: 인자 2개 → NVL2 는 3개 필수
SELECT NVL2(bonus, '대상자') FROM employees;

-- ✅ 인자 2개짜리는 NVL 사용
SELECT NVL(bonus, '제외') FROM employees;

-- ✅ NVL2 는 무조건 3개
SELECT NVL2(bonus, '대상자', '제외') FROM employees;
```

|함수|인자 수|비고|
|---|:-:|---|
|`NVL(컬럼, 대체값)`|**2개**|NULL 이면 대체값|
|`NVL2(컬럼, 값A, 값B)`|**3개 필수**|NULL 아니면 A, NULL 이면 B|
|`COALESCE(값1, 값2, ...)`|**2개 이상**|처음으로 NULL 아닌 값|
|`NULLIF(값1, 값2)`|**2개**|같으면 NULL, 다르면 값1|

---

---

# ⑤ 함수 한눈에 비교

|함수|DB|인자 수|핵심 동작|
|---|---|:-:|---|
|`COALESCE`|전체|2개 이상|처음으로 NULL 아닌 값 반환. 전부 NULL 이면 NULL|
|`NVL`|Oracle|2개|NULL 이면 대체값|
|`ISNULL`|SQL Server|2개|NULL 이면 대체값|
|`NVL2`|Oracle|**3개 필수**|NULL 아니면 A, NULL 이면 B|
|`NULLIF`|전체|2개|같으면 NULL, 다르면 첫 번째 값|

---

---

# SQLD 빈출 함정

## COALESCE 타입 불일치 에러

```sql
-- ❌ age 는 숫자(INT), '비밀' 은 문자 → 타입 에러
SELECT COALESCE(age, '비밀') FROM users;

-- ✅ 타입 통일 필수
SELECT COALESCE(age::VARCHAR, '비밀') FROM users;        -- PostgreSQL
SELECT COALESCE(CAST(age AS VARCHAR), '비밀') FROM users; -- 표준
```

## AVG + NULL 함정

```sql
-- 데이터: 100점, 50점, NULL (미응시자)
AVG(score)                →  (100+50) / 2명 = 75점  ← NULL 제외해서 평균 높아짐
AVG(COALESCE(score, 0))   →  (100+50+0) / 3명 = 50점 ← 정확한 전체 평균
```

## NVL vs ISNULL vs IS NULL 혼동 주의

|표현|역할|사용 위치|
|---|---|---|
|`NVL(컬럼, 0)`|NULL 을 0 으로 **바꾸는 함수**|SELECT, WHERE 어디든|
|`ISNULL(컬럼, 0)`|NULL 을 0 으로 **바꾸는 함수** (SQL Server)|SELECT, WHERE 어디든|
|`컬럼 IS NULL`|NULL 인지 **검색하는 조건 연산자**|WHERE 절|

```sql
-- 완전히 다른 용도
SELECT NVL(bonus, 0) FROM emp;          -- bonus 가 NULL 이면 0 으로 출력
SELECT * FROM emp WHERE bonus IS NULL;  -- bonus 가 NULL 인 행 검색
```

---
