---
aliases:
  - DISTINCT
  - 중복제거
  - Unique
  - COUNT(DISTINCT 컬럼)
tags:
  - SQL
related:
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Window_Functions]]"
---
# SQL DISTINCT vs GROUP BY

## 개념 한 줄 요약

> **"종류(목록)만 알고 싶으면 `DISTINCT`, 묶어서 통계를 내려면 `GROUP BY`, 그룹 안에서 고유한 개수를 세려면 `COUNT(DISTINCT)`, 그룹별 특정 행 1건만 뽑으려면 `DISTINCT ON`을 쓴다."**

---

## 언제 무엇을 쓰나?

|상황|사용|
|---|---|
|"어떤 카테고리가 있지?" — 목록만 뽑기|`SELECT DISTINCT`|
|"카테고리별로 몇 개씩이지?" — 통계 내기|`GROUP BY`|
|"오늘 방문자 수(중복 제거)" — 순수 건수 세기|`COUNT(DISTINCT)`|
|"열차별 가장 최신 상태 1건만" — 그룹별 특정 행|`DISTINCT ON` ⭐ PostgreSQL|

**실무 예시:**

- 쇼핑몰 대분류·소분류 목록 만들기 → `SELECT DISTINCT`
- DAU(일일 활성 사용자) 계산: 철수가 10번 접속해도 1명으로 셈 → `COUNT(DISTINCT user_id)`
- 60초마다 쌓이는 열차 데이터에서 열차별 최신 1건만 → `DISTINCT ON (trn_no)`

---

## 4가지 문법 비교표

|구분|`SELECT DISTINCT`|`GROUP BY`|`COUNT(DISTINCT)`|`DISTINCT ON`|
|---|---|---|---|---|
|목적|목록·메뉴판 만들기|데이터 묶기 (통계용)|고유값 개수 세기|그룹별 특정 행 1건|
|사용 위치|`SELECT` 바로 뒤|`FROM` / `WHERE` 뒤|`SELECT` · `HAVING` 절 내부|`SELECT` 바로 뒤|
|작동 원리|행 전체 중복 제거|기준으로 파티셔닝|그룹 내 중복 뺀 개수|ORDER BY 기준 첫 행만|
|DB 지원|표준 SQL|표준 SQL|표준 SQL|PostgreSQL 전용 ⚠️|

---

---

# ① SELECT DISTINCT — 목록 추출

## 기본 사용

```sql
-- 배송이 갔던 시/도 · 구/군 목록을 중복 없이 확인
SELECT DISTINCT city, district
FROM orders
ORDER BY city, district;
```

---

## ⭐ 핵심 원리 — 여러 컬럼일 때 중복 판단 기준

> **DISTINCT 는 뒤에 나열된 컬럼 값이 `모두` 동일한 행만 중복으로 처리한다.**

컬럼 하나하나가 아니라 **나열된 컬럼 전체를 하나의 세트** 로 묶어서 중복 여부를 판단한다.

```sql
SELECT DISTINCT name, city FROM members;
```

|name|city|결과|
|---|---|---|
|철수|서울|✅ 출력|
|철수|부산|✅ 출력 — name은 같지만 city가 다름 → **다른 행으로 취급**|
|철수|서울|❌ 제거 — name·city 모두 동일 → **중복으로 제거**|
|영희|서울|✅ 출력|

> **결론:** `(철수, 서울)` 과 `(철수, 부산)` 은 name 은 같지만 city 가 다르므로 서로 다른 행이다. **A 컬럼만 완벽하게 중복 제거하고 싶다면** 뒤에 B 컬럼을 같이 SELECT 하면 안 된다.

```sql
SELECT DISTINCT name FROM members;        -- name 기준으로만 중복 제거 ✅
SELECT DISTINCT name, city FROM members;  -- (name + city) 세트 기준으로 중복 제거
```

---

## DISTINCT 문법 O/X 체크리스트

```sql
-- ✅ 올바른 위치: SELECT 바로 뒤에 한 번만
SELECT DISTINCT col1, col2 FROM table;

-- ✅ 올바른 위치: 집계함수 안에
SELECT COUNT(DISTINCT col) FROM table;

-- ❌ 절대 불가: 특정 컬럼 하나에만 씌우기
SELECT col1, DISTINCT(col2) FROM table;
-- DISTINCT는 엑셀 함수처럼 컬럼 하나에만 붙일 수 없다.
-- SELECT 절에서는 무조건 맨 앞에 한 번만!
```

---

## DISTINCT (A || B) — 문자열 결합 중복 제거

`||` 는 Oracle·PostgreSQL 에서 쓰는 문자열 결합 연산자다. 두 컬럼을 하나로 합친 뒤 중복을 제거하는 테크닉이다.

```sql
-- 작동 방식: '홍' + '길동' → '홍길동' 으로 합친 뒤 그 덩어리의 중복을 제거
COUNT(DISTINCT colA || colB)
```

---

---

# ② GROUP BY — 그룹화 후 통계

집계가 필요 없고 단순히 "어떤 조합이 있는지"만 궁금할 때는 `DISTINCT` 가 더 직관적이다. `GROUP BY` 는 **그룹화된 통계** 가 필요할 때 써야 한다.

```sql
-- 지역별 주문 건수 집계
SELECT city, COUNT(*) AS order_cnt
FROM orders
GROUP BY city
ORDER BY order_cnt DESC;
```

---

---

# ③ COUNT(DISTINCT) — 고유 개수 세기

> **"DISTINCT 가 집계 함수 안에 들어갈 때 진짜 파괴력이 살아난다."**

```sql
-- 서로 다른 펭귄 종을 2종 이상 보유한 섬만 찾아라
SELECT island, COUNT(DISTINCT species) AS species_cnt
FROM penguins
GROUP BY island
HAVING COUNT(DISTINCT species) >= 2;
```

**동작 흐름:**

```
1. island 별로 데이터를 묶는다 (GROUP BY)
2. 그 섬에 사는 펭귄 중 이름이 같은 종은 1번만 카운트 (DISTINCT)
3. 고유 종 수가 2 이상인 섬만 필터링 (HAVING)
```

---

---

# ④ DISTINCT ON — 그룹별 특정 행 1건 ⭐ PostgreSQL 전용

## 개념

```
SELECT DISTINCT 는 중복 행 전체를 제거
DISTINCT ON 은 특정 컬럼 기준으로 그룹을 나누고
              ORDER BY 기준으로 정렬했을 때 첫 번째 행 1건만 남김

→ "각 그룹에서 가장 최신(또는 최고/최저) 행 1건만 뽑을 때" 사용
```

## 문법

```sql
SELECT DISTINCT ON (중복제거_기준_컬럼)
    컬럼1, 컬럼2, ...
FROM 테이블
ORDER BY 중복제거_기준_컬럼, 정렬기준 DESC;
--         ↑ DISTINCT ON 컬럼이 ORDER BY 첫 번째에 반드시 와야 함
```

## 예시 — 열차별 최신 상태 1건만

```sql
-- 60초마다 쌓이는 train_realtime 에서 열차별 가장 최신 행만
SELECT DISTINCT ON (trn_no)
    trn_no, mrnt_nm, status, progress_pct, created_at
FROM train_realtime
ORDER BY trn_no, created_at DESC;
--                ↑ created_at 내림차순 → 가장 최신 행이 첫 번째
```

```
동작 흐름:
  1. trn_no 기준으로 그룹을 나눔
  2. 각 그룹 안에서 created_at DESC 로 정렬
  3. 정렬된 그룹에서 첫 번째 행(= 가장 최신)만 남김
  4. 나머지 중복 행 제거
```

## DISTINCT ON vs 다른 방법 비교

```sql
-- ① DISTINCT ON (PostgreSQL 전용, 가장 간결)
SELECT DISTINCT ON (trn_no) trn_no, status, created_at
FROM train_realtime
ORDER BY trn_no, created_at DESC;

-- ② ROW_NUMBER (표준 SQL, 모든 DB 지원)
SELECT trn_no, status, created_at
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY trn_no ORDER BY created_at DESC) AS rn
    FROM train_realtime
) t
WHERE rn = 1;

-- ③ 서브쿼리 MAX (표준 SQL, 단순하지만 타이 브레이킹 불가)
SELECT t.*
FROM train_realtime t
JOIN (
    SELECT trn_no, MAX(created_at) AS latest
    FROM train_realtime
    GROUP BY trn_no
) m ON t.trn_no = m.trn_no AND t.created_at = m.latest;
```

```
성능 비교:
  DISTINCT ON  → PostgreSQL 옵티마이저가 잘 최적화 / 코드 간결
  ROW_NUMBER   → 표준 SQL / 다른 DB 에서도 동작 / 코드 길어짐
  MAX 서브쿼리 → 동시에 같은 시각 데이터 있으면 중복 발생 가능

PostgreSQL 쓴다면 DISTINCT ON 이 가장 간결하고 직관적
```

```
DB 별 지원 현황:
  PostgreSQL  DISTINCT ON 지원 ✅
  MySQL       DISTINCT ON 없음 → ROW_NUMBER() 로 대체
  Oracle      DISTINCT ON 없음 → ROW_NUMBER() 로 대체
  SQL Server  DISTINCT ON 없음 → ROW_NUMBER() 로 대체
  SQLite      DISTINCT ON 없음 → ROW_NUMBER() 로 대체

→ DISTINCT ON 은 PostgreSQL 전용
  다른 DB 에서는 ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...) 가 표준 대체 방법
```

>ROW_NUMBER() [[SQL_Window_Functions#ROW_NUMBER() — 무조건 고유한 번호|ROW_NUMBER()]] 참고

## ⚠️ 주의사항

```sql
-- ❌ ORDER BY 첫 번째가 DISTINCT ON 컬럼이 아닐 때
SELECT DISTINCT ON (trn_no) trn_no, status
FROM train_realtime
ORDER BY created_at DESC;   -- 에러: trn_no 가 ORDER BY 첫 번째에 없음

-- ✅ DISTINCT ON 컬럼이 ORDER BY 첫 번째에 와야 함
SELECT DISTINCT ON (trn_no) trn_no, status
FROM train_realtime
ORDER BY trn_no, created_at DESC;
```

---

---

# 핵심 정리 — 언제 뭘 쓸지 한눈에

```
단순 목록이 필요해?
  └── YES → SELECT DISTINCT

집계(합계·평균·개수)가 필요해?
  └── YES → GROUP BY

그룹 안에서 중복 제거한 개수가 필요해?
  └── YES → COUNT(DISTINCT col)

특정 컬럼 하나만 중복 제거하고 싶어?
  └── 뒤에 다른 컬럼 붙이지 말고 SELECT DISTINCT 해당컬럼 만 쓸 것

그룹별 최신(최고/최저) 행 1건만 뽑고 싶어? (PostgreSQL)
  └── YES → DISTINCT ON (기준컬럼) + ORDER BY 기준컬럼, 정렬기준
```

---

