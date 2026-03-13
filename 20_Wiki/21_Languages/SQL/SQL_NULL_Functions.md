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
  - "[[SQL_Date_Functions]]"
  - "[[SQL_Numeric_Functions]]"
---
# SQL_NULL_Functions — NULL 처리 함수

## 한 줄 요약

```
연산을 집어삼키는 블랙홀(NULL) 을
안전한 값으로 메워주는 심폐소생술
```

---

---

# 왜 필요한가?

```
① 블랙홀 연산 방지
   100 + NULL = NULL
   보너스가 NULL 인 직원의 총 포인트가 통째로 NULL → 대참사

② 통계 왜곡 방지
   AVG() 는 NULL 을 분모에서 제외
   → 0점 처리해야 할 유저가 빠지면 평균이 비정상적으로 높아짐

③ UI/UX 개선
   화면에 'null' 노출 방지 → '미입력', '0원' 으로 변환
```

---

---

# 함수 한눈에 비교

|함수|DB|인자 수|핵심 동작|
|---|---|--:|---|
|`COALESCE`|전체 (표준)|2개 이상|처음으로 NULL 아닌 값 반환|
|`NVL`|Oracle|2개|NULL 이면 대체값|
|`ISNULL`|SQL Server|2개|NULL 이면 대체값|
|`NVL2`|Oracle|**3개 필수**|NULL 아니면 A, NULL 이면 B|
|`NULLIF`|전체 (표준)|2개|같으면 NULL, 다르면 첫 번째 값|

---

---

# ① COALESCE — 표준 SQL 만능 NULL 처리

```
COALESCE(값1, 값2, 값3, ...)
  왼쪽부터 순서대로 보고 처음으로 NULL 이 아닌 값 반환
  전부 NULL 이면 → NULL 반환

Oracle · PostgreSQL · SQL Server 모두 지원
```

```sql
SELECT COALESCE(NULL, NULL, 'C', 'D');   -- 'C'  (처음으로 NULL 아닌 값)
SELECT COALESCE('A',  NULL, 'C');        -- 'A'  (첫 번째가 이미 NULL 아님)
SELECT COALESCE(NULL, NULL);             -- NULL (전부 NULL 이면 NULL)
```

```
⚠️ 전부 NULL 이면 결과도 NULL
   마지막 인자를 항상 고정 대체값으로 써야 안전

   COALESCE(값1, 값2, '기본값')  ← 마지막 = NULL 방어막
```

```sql
-- 실전: 연락처 우선순위 (핸드폰 → 집전화 → 없음)
SELECT COALESCE(phone_mobile, phone_home, '연락처 없음') FROM users;

-- LEFT JOIN NULL 처리
SELECT
    a.name,
    COALESCE(b.amount, 0) AS amount   -- 매칭 없으면 0
FROM customers a
LEFT JOIN orders b ON a.id = b.customer_id;
```

---

---

# ② NVL / ISNULL — DB 전용 2인자 버전

```
딱 2개: (검사할 컬럼, NULL 일 때 대체값)
  NULL 아님 → 원래 값 그대로
  NULL 임   → 대체값 반환

PostgreSQL 은 NVL 없음 → COALESCE 사용
```

```sql
-- Oracle
SELECT NVL(bonus, 0)        FROM salary;   -- NULL 이면 0
SELECT NVL(name,  '무명')   FROM users;    -- NULL 이면 '무명'

-- SQL Server
SELECT ISNULL(bonus, 0)     FROM salary;
SELECT ISNULL(name,  '무명') FROM users;
```

---

---

# ③ NVL2 — Oracle 전용 3단 콤보

```
NVL2(검사할_컬럼, NULL이_아닐_때, NULL일_때)
  NULL 아님 → 두 번째 인자
  NULL 임   → 세 번째 인자

IF-ELSE 와 동일
Oracle 전용 — PostgreSQL/MSSQL 은 CASE WHEN 사용
```

```sql
SELECT NVL2(bonus, '대상자', '제외') FROM employees;
-- bonus 있으면 '대상자', NULL 이면 '제외'
```

```
⚠️ 인자 수 엄격히 구분

  NVL(컬럼, 대체값)        인자 2개  → NULL 이면 대체값
  NVL2(컬럼, 값A, 값B)     인자 3개  → NULL 아니면 A, NULL 이면 B
  NVL2(bonus, '대상자')    ❌ 에러   → 3개 필수
```

---

---

# ④ NULLIF — 같으면 NULL 로 (0 나눗셈 방지)

```
NULLIF(값1, 값2)
  값1 = 값2  →  NULL 반환
  값1 ≠ 값2  →  값1 그대로 반환

주 용도: 분모가 0 일 때 Division by Zero 에러 방지
```

```sql
SELECT NULLIF('A', 'A');   -- NULL  (같으니까 폭파)
SELECT NULLIF('A', 'B');   -- 'A'   (다르니까 그대로)
SELECT NULLIF(0, 0);       -- NULL
SELECT NULLIF(5, 0);       -- 5

-- 실전: 0 나눗셈 방지
SELECT amount / NULLIF(total_count, 0) FROM sales;
-- total_count = 0  → NULL → 에러 없이 결과도 NULL

-- 진행률 계산 (출발=도착 이면 분모 0 방지)
/ NULLIF(EXTRACT(EPOCH FROM (plan_arr::TIME - plan_dep::TIME)), 0)
```

```
NULLIF vs COALESCE 역할 구분:
  NULLIF    → 특정 값을 NULL 로 만들 때  (0 제거 등)
  COALESCE  → NULL 을 다른 값으로 채울 때
```

---

---

# SQLD 빈출 함정

## ① COALESCE 타입 불일치 에러

```sql
-- ❌ age(INT) 와 '비밀'(VARCHAR) 타입 불일치 → 에러
SELECT COALESCE(age, '비밀') FROM users;

-- ✅ 타입 통일 필수
SELECT COALESCE(age::VARCHAR, '비밀') FROM users;          -- PostgreSQL
SELECT COALESCE(CAST(age AS VARCHAR), '비밀') FROM users;  -- 표준
```

## ② AVG + NULL 함정

```sql
-- 데이터: 100점, 50점, NULL (미응시자 → 0점 처리해야 함)

AVG(score)
-- (100 + 50) / 2명 = 75점  ← NULL 제외되어 평균 뻥튀기 ❌

AVG(COALESCE(score, 0))
-- (100 + 50 + 0) / 3명 = 50점  ← 정확한 전체 평균 ✅
```

## ③ NVL vs IS NULL — 완전히 다른 용도

```
NVL(컬럼, 0)    → NULL 을 0 으로 바꾸는 함수   (SELECT, WHERE 어디든)
컬럼 IS NULL    → NULL 인지 검색하는 조건       (WHERE 절)
```

```sql
SELECT NVL(bonus, 0) FROM emp;           -- NULL 이면 0 으로 출력
SELECT * FROM emp WHERE bonus IS NULL;   -- NULL 인 행 검색

-- ❌ 절대 이렇게 쓰면 안 됨
SELECT * FROM emp WHERE bonus = NULL;    -- 항상 0건 반환
-- NULL = NULL 은 FALSE/UNKNOWN → IS NULL 만 사용
```

## ④ NULL 비교 연산 핵심 원칙

```
NULL 은 어떤 값과도 같지 않음 (자기 자신과도)

NULL = NULL    → FALSE / UNKNOWN
NULL + 숫자    → NULL
NULL 과 비교   → NULL

검색:   WHERE 컬럼 IS NULL  /  IS NOT NULL
비교:   COALESCE(컬럼, 0) = 0
집계:   COUNT(*)    → NULL 포함 전체 행 수
        COUNT(컬럼) → NULL 제외한 행 수
```