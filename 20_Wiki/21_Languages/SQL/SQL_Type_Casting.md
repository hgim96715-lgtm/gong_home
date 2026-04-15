---
aliases:
  - 형변환
  - TO_CHAR
  - TO_DATE
  - 명시적 형변환
  - 암시적 형변환
tags:
  - SQL
related:
  - "[[SQL_Data_Types]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_String_Functions]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_Numeric_Functions]]"
---
# SQL 형변환 (Type Casting) — 명시적 · 암시적

## 개념 한 줄 요약

> **"숫자를 문자로, 문자를 날짜로! 데이터의 타입을 상황에 맞게 변환하는 작업."** 문자열 `'123'` 과 숫자 `123` 은 눈으로 보기엔 같지만 DB 입장에선 완전히 다른 종족이다. 이 둘을 연산하거나 비교하려면 타입을 통일시켜야 한다.

---

---

# ① 형변환의 2가지 방식

|방식|설명|권장 여부|
|---|---|:-:|
|**암시적 형변환**|DB 가 자동으로 타입을 변환|❌ 비권장|
|**명시적 형변환**|사용자가 변환 함수를 직접 명시|✅ 권장|

---

## A. 암시적 형변환 (Implicit Casting) — DB 가 눈치껏

사용자가 명령하지 않아도 DB 엔진이 **자동으로 타입을 변환**해서 처리하는 방식.

```sql
SELECT '100' + 200;   -- 문자 '100' → 숫자 100 으로 자동 변환 → 결과: 300
```

> ⚠️ **치명적 단점:** DB 가 몰래 타입을 변환하느라 인덱스(Index) 를 타지 못하고 **풀 테이블 스캔(Full Table Scan)** 이 발생할 수 있다 → 성능 저하의 주범.

---

## B. 명시적 형변환 (Explicit Casting) — 직접 지정

`CAST` · `TO_CHAR` 같은 변환 함수를 써서 **"이 타입으로 바꿔!"** 라고 명확하게 지시하는 방식.

```sql
SELECT CAST('123' AS NUMBER) + 200;   -- 명시적으로 숫자로 변환 후 연산
```

> ✅ 실무에서는 100% 명시적 형변환을 권장한다.

---

---

# ② 암시적 형변환 방향 규칙 — Oracle vs PostgreSQL ⭐️

> **암시적 형변환은 방향이 있다.** "어느 쪽 컬럼이 문자형이고, 어느 쪽이 숫자형이냐" 에 따라 에러 여부가 달라진다.

## Oracle 의 방향 규칙

```
규칙: 숫자형 컬럼 ↔ 문자형 데이터  →  자동 변환 ✅ (에러 없음)
      문자형 컬럼 ↔ 숫자형 데이터  →  에러 발생 ❌
```

### 케이스 1 — 숫자형 컬럼에 문자형 비교값 → 에러 없음 ✅

```sql
-- salary 컬럼이 NUMBER 타입이라고 가정
SELECT * FROM 사원 WHERE salary = '3000';
--                                ↑ 문자형 '3000'

-- Oracle 의 처리:
-- 문자 '3000' 을 숫자 3000 으로 자동 변환
-- WHERE salary = 3000 으로 처리 → 정상 실행
```

```
흐름: 문자 '3000' → (자동 변환) → 숫자 3000 → 숫자 컬럼과 비교 ✅
```

### 케이스 2 — 문자형 컬럼에 숫자형 비교값 → 에러 발생 ❌

```sql
-- user_id 컬럼이 VARCHAR2 타입이라고 가정
SELECT * FROM 사원 WHERE user_id = 12345;
--                                ↑ 숫자형 12345

-- Oracle 의 처리:
-- 문자형 컬럼의 모든 값을 숫자로 변환 시도
-- 만약 'A001' 같은 숫자로 변환 불가능한 값이 있으면 → ORA-01722: invalid number 에러!
```

```
흐름: 문자 컬럼의 모든 값을 숫자로 변환 시도 → 변환 불가 값 만나면 에러 ❌

ORA-01722: invalid number
```

> **왜 방향이 다른가?** Oracle 은 비교 시 "더 정밀한 타입(숫자) 기준으로 맞추려는" 경향이 있다. 숫자 → 문자 방향 변환보다 문자 → 숫자 방향 변환을 먼저 시도하는데, 문자형 컬럼 전체를 숫자로 바꾸려다 변환 불가 값이 있으면 에러가 터진다.

---

## PostgreSQL 의 방향 규칙 — 더 엄격함

```
규칙: 타입이 다르면 기본적으로 에러. 암시적 변환 범위가 Oracle 보다 훨씬 좁다.
```

```sql
-- PostgreSQL: 숫자형 컬럼에 문자형 비교값
SELECT * FROM 사원 WHERE salary = '3000';
-- 🐘 PostgreSQL: '3000' (text) 과 salary (integer) 타입 불일치 → 에러!
-- ERROR: operator does not exist: integer = text

-- PostgreSQL: 문자형 컬럼에 숫자형 비교값
SELECT * FROM 사원 WHERE user_id = 12345;
-- 🐘 PostgreSQL: user_id (varchar) 와 12345 (integer) 타입 불일치 → 에러!
-- ERROR: operator does not exist: character varying = integer
```

> **PostgreSQL 은 숫자 리터럴에 대해 타입 추론을 일부 허용하는 경우도 있지만, 기본적으로 타입이 다르면 명시적 변환을 요구한다.**

---

## Oracle vs PostgreSQL 암시적 형변환 비교

|상황|Oracle|PostgreSQL|
|---|:-:|:-:|
|숫자형 컬럼 = `'3000'` (문자 리터럴)|✅ 자동 변환|❌ 에러|
|문자형 컬럼 = `12345` (숫자 리터럴)|❌ 에러 가능 (변환 불가 값 있으면)|❌ 에러|
|`'100' + 200`|✅ 자동 변환 → 300|❌ 에러|
|`SYSDATE + 1` (날짜 + 숫자)|✅ 1일 후 날짜 반환|❌ 명시적 `INTERVAL` 필요|

> **결론:** Oracle 은 암시적 변환이 관대하지만 방향에 따라 에러가 날 수 있고, PostgreSQL 은 처음부터 엄격하게 타입 일치를 요구한다. **두 DB 모두 명시적 형변환을 쓰는 것이 안전하다.**

---

---

# ③ 명시적 형변환 함수

## A. CAST() — 표준 SQL (모든 DB 공통)

```sql
SELECT CAST('2026-02-24' AS DATE);    -- 문자 → 날짜
SELECT CAST(123.45 AS INT);           -- 실수 → 정수 (결과: 123)
SELECT CAST(123 AS VARCHAR);          -- 숫자 → 문자
```

---

## B. PostgreSQL 전용 — `::` 단축 문법

```sql
SELECT '2026-02-24'::DATE;      -- 문자 → 날짜
SELECT 123.45::INT;             -- 실수 → 정수
SELECT 123::VARCHAR;            -- 숫자 → 문자
SELECT '3000'::INTEGER + 200;   -- 문자를 숫자로 변환 후 연산 → 3200
```

---

## C. Oracle 전용 — TO_ 패밀리 (SQLD 핵심 3대장)

|함수|방향|예시|
|---|---|---|
|`TO_CHAR(값, '포맷')`|숫자·날짜 → **문자**|`TO_CHAR(SYSDATE, 'YYYY-MM-DD')`|
|`TO_NUMBER('문자열')`|문자 → **숫자**|`TO_NUMBER('12345')`|
|`TO_DATE('문자열', '포맷')`|문자 → **날짜**|`TO_DATE('20260224', 'YYYYMMDD')`|

```sql
-- 🔴 Oracle (SQLD 단골 출제)
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD') FROM DUAL;      -- 날짜 → 문자
SELECT TO_NUMBER('12345') + 100 FROM DUAL;            -- 문자 → 숫자 (결과: 12445)
SELECT TO_DATE('20260224', 'YYYYMMDD') FROM DUAL;     -- 문자 → 날짜
```

---

## D. SQL Server 전용 — CONVERT()

```sql
-- 문법: CONVERT(바꿀_타입, 대상, [스타일코드])
SELECT CONVERT(INT, '123');               -- 문자 → 정수
SELECT CONVERT(VARCHAR, GETDATE(), 23);   -- 날짜 → 문자 (스타일 23 = YYYY-MM-DD)
```

---

---

# ④ 자주 하는 실수 2가지

## 실수 1 — 인덱스 박살 내기 (가장 흔한 실무 장애)

`user_id` 컬럼이 `VARCHAR` 타입일 때:

```sql
-- ❌ 나쁜 예: 숫자로 검색 → 암시적 형변환 발생
SELECT * FROM users WHERE user_id = 12345;
-- DB 가 테이블의 모든 user_id 를 숫자로 변환 시도
-- → 인덱스 사용 불가 → 풀 테이블 스캔 → 서버 뻗음

-- ✅ 좋은 예: 타입을 맞춰서 검색
SELECT * FROM users WHERE user_id = '12345';
-- 인덱스 정상 사용
```

## 실수 2 — 날짜 포맷 안 알려주기

```sql
-- ❌ 포맷 미지정 → Oracle 에서 에러 위험
SELECT TO_DATE('2026/02/24') FROM DUAL;

-- ✅ 포맷 명시
SELECT TO_DATE('2026/02/24', 'YYYY/MM/DD') FROM DUAL;
```

---

---

# 핵심 요약 카드

```
┌──────────────────────────────────────────────────────────────┐
│  암시적 형변환 방향 규칙 (Oracle):                             │
│  숫자형 컬럼 = '문자' → 자동 변환 ✅ (에러 없음)              │
│  문자형 컬럼 = 숫자   → 에러 가능 ❌ (변환 불가 값 있으면)    │
│                                                              │
│  PostgreSQL 는 더 엄격:                                       │
│  타입이 다르면 기본적으로 에러 → 명시적 변환 필요              │
│                                                              │
│  명시적 형변환 함수:                                          │
│  표준 SQL    → CAST(값 AS 타입)                              │
│  PostgreSQL → 값::타입  (단축 문법)                           │
│  Oracle      → TO_CHAR / TO_NUMBER / TO_DATE                │
│  SQL Server  → CONVERT(타입, 값, 스타일)                     │
│                                                              │
│  암시적 형변환의 문제:                                         │
│  인덱스 무력화 → 풀 테이블 스캔 → 성능 저하                   │
│  실무에서는 항상 명시적 형변환 사용                            │
└──────────────────────────────────────────────────────────────┘
```

---
