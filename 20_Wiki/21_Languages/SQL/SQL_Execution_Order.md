---
aliases:
  - SQL 실행순서
  - 작성순서
  - Parsing Order
  - Query Order
  - SQL 작동원리
tags:
  - SQL
related:
  - "[[HAVING_vs_WHERE]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_Window_Functions]]"
---
# SQL_Execution_Order — SQL 실행 순서

## 한 줄 요약

```
작성 순서(Writing) ≠ 실행 순서(Execution)
이 괴리 때문에 수많은 에러 발생
```

---

---

# ① 실행 순서 한눈에

|실행 순서|절|작성 순서|역할|
|---|---|---|---|
|1|`FROM` / `JOIN`|2|테이블 가져오기 (재료 확보)|
|2|`WHERE`|3|행 필터링 1차|
|3|`GROUP BY`|4|그룹핑|
|4|`HAVING`|5|그룹 필터링 2차|
|**5**|**`SELECT`**|**1**|**컬럼 추출 + 별칭(AS) 생성**|
|5.5|`Window 함수`|-|SELECT 계산 중|
|6|`ORDER BY`|6|정렬|
|7|`LIMIT`|7|건수 제한|

```
암기:
  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
  "프웨그해 셀오리"
```

---

---

# ② 단계별 설명

## Step 1. FROM / JOIN — 재료 확보

```
테이블을 메모리에 올림
JOIN 있으면 두 테이블을 붙여서 가상의 큰 테이블 생성

⚠️ 아직 별칭(AS) 이나 SELECT 계산이 안 된 상태
```

## Step 2. WHERE — 행 필터링 1차

```
조건에 맞지 않는 행(Row) 제거
집계 전 필터링 → 개별 행 기준

⚠️ 에러 포인트:
  SELECT 에서 만든 별칭을 WHERE 에서 쓰면 에러
  → 아직 SELECT 가 실행되지 않아서 별칭이 존재하지 않음
```

```sql
-- ❌ 에러 — WHERE 에서 SELECT 별칭 사용
SELECT amount * 100 AS total_amount
FROM orders
WHERE total_amount > 5000;
-- ERROR: column "total_amount" does not exist
-- 이유: WHERE(2번) 실행 시점에 SELECT(5번) 별칭이 아직 없음

-- ✅ 해결 — WHERE 에서는 원래 표현식 사용
SELECT amount * 100 AS total_amount
FROM orders
WHERE amount * 100 > 5000;
```

## Step 3. GROUP BY — 그룹핑

```
살아남은 행을 기준 컬럼으로 묶음
이후 SELECT 에서 집계 함수(SUM/COUNT/AVG) 사용 가능
```

## Step 4. HAVING — 그룹 필터링 2차

```
묶인 그룹의 집계 결과로 필터링
WHERE 와 다르게 집계 함수 사용 가능

WHERE  개별 행 필터 (집계 전)
HAVING 그룹 필터  (집계 후)
```

```sql
-- WHERE vs HAVING
SELECT region, COUNT(*) AS cnt
FROM hospitals
WHERE duty_eryn = '1'          -- ① 운영 중인 병원만 (행 필터)
GROUP BY region
HAVING COUNT(*) >= 10;         -- ② 10개 이상인 지역만 (그룹 필터)
```

## Step 5. SELECT — 컬럼 추출 + 별칭 생성

```
이제야 컬럼을 뽑고 집계 함수 계산
AS 로 별칭 생성
이 시점부터 별칭이 존재함
```

## Step 5.5. Window 함수 — SELECT 계산 중 ⭐️

```
Window 함수는 SELECT 절 계산 중에 실행
GROUP BY / HAVING 이후 단계

중요:
  Window 함수는 GROUP BY 와 같이 쓸 수 없음
  GROUP BY 는 행을 합쳐버리지만
  Window 함수는 각 행을 유지하면서 계산
```

```sql
-- ❌ 에러 — GROUP BY 와 Window 함수 동시 사용
SELECT
    region,
    SUM(hvec) OVER (PARTITION BY region) AS region_total
FROM er_realtime
GROUP BY region;
-- ERROR: window 함수는 집계 이후 단계에서 동작
--        GROUP BY 로 이미 행이 합쳐진 상태 → Window 함수 적용 불가

-- ✅ 해결 방법 1 — GROUP BY 없이 Window 함수만 사용
SELECT
    hpid,
    region,
    hvec,
    SUM(hvec) OVER (PARTITION BY region) AS region_total
FROM er_realtime;

-- ✅ 해결 방법 2 — 서브쿼리로 분리
SELECT
    region,
    SUM(hvec) AS total,
    SUM(SUM(hvec)) OVER () AS grand_total
FROM er_realtime
GROUP BY region;
-- GROUP BY 먼저 → 그 결과에 Window 함수 적용
```

```
Window 함수 실행 시점 요약:
  GROUP BY → HAVING → SELECT 계산 시작 → Window 함수 계산 → ORDER BY

  Window 함수는 GROUP BY 결과 위에서 동작
  raw 데이터에 GROUP BY + Window 함수 동시 적용 불가
```

## Step 6. ORDER BY — 정렬

```
SELECT 이후에 실행
→ SELECT 에서 만든 별칭 사용 가능 ✅
```

```sql
SELECT amount * 100 AS total_amount
FROM orders
ORDER BY total_amount DESC;   -- ✅ 별칭 사용 가능
```

## Step 7. LIMIT — 건수 제한

```
정렬된 결과에서 N개만 반환
```

---

---

# ③ 별칭(AS) 사용 가능 시점 정리

```
WHERE    ❌  SELECT 아직 실행 안 됨
HAVING   ❌  SELECT 아직 실행 안 됨 (집계 함수 직접 써야 함)
ORDER BY ✅  SELECT 이후 → 별칭 사용 가능
```

```sql
-- 요약 예시
SELECT
    region,
    COUNT(*) AS cnt,                        -- 별칭 생성
    AVG(hvec) AS avg_beds
FROM er_realtime
WHERE duty_eryn = '1'                       -- ❌ cnt, avg_beds 못 씀
GROUP BY region
HAVING COUNT(*) >= 5                        -- ❌ cnt 못 씀 → COUNT(*) 직접 써야
ORDER BY avg_beds DESC                      -- ✅ avg_beds 사용 가능
LIMIT 10;
```

---

---

# ④ 자주 하는 실수 요약

|실수|원인|해결|
|---|---|---|
|`WHERE` 에서 SELECT 별칭 사용|WHERE(2) 는 SELECT(5) 전|WHERE 에 원래 표현식 사용|
|`HAVING` 에서 SELECT 별칭 사용|HAVING(4) 는 SELECT(5) 전|HAVING 에 집계 함수 직접 사용|
|`GROUP BY` + `Window 함수` 동시|Window 함수는 집계 이후|서브쿼리로 분리|
|`WHERE` 에 집계 함수 사용|WHERE 는 집계 전 단계|`HAVING` 으로 이동|