---
aliases:
  - 윈도우함수
  - Window Functions
  - 순위함수
  - 집계함수
  - 행순서함수
  - 비율함수
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[PyFlink_Windows]]"
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_CASE_WHEN]]"
---
# SQL_Window_Functions — 윈도우 함수

## 한 줄 요약

```
GROUP BY 처럼 행을 압축하지 않고
원본 행을 유지한 채 계산 결과를 옆에 붙여주는 함수
```

## GROUP BY vs PARTITION BY

```
GROUP BY     → 행을 합쳐서 요약 (행 수 줄어듦)
PARTITION BY → 원본 유지 + 계산값만 추가 (행 수 그대로)
```

```sql
-- GROUP BY: 부서별 1줄만 남음
SELECT 부서, AVG(급여) FROM 사원 GROUP BY 부서;

-- PARTITION BY: 원본 행 유지 + 부서평균 컬럼 추가
SELECT 사원명, 부서, 급여,
       AVG(급여) OVER (PARTITION BY 부서) AS 부서평균
FROM 사원;
```

## 기본 문법

```sql
함수명() OVER (
    PARTITION BY 컬럼   -- 그룹 나누기 (선택)
    ORDER BY 컬럼       -- 정렬 기준   (필수 여부는 함수마다 다름)
    ROWS BETWEEN 시작 AND 끝  -- 범위 지정 (선택)
)
```

---

---

# ① 순위 함수 ⭐️

|함수|동점 처리|출력 예시|
|---|---|---|
|`ROW_NUMBER()`|무조건 고유 번호|1, 2, 3, 4|
|`RANK()`|같은 등수, 번호 건너뜀|1, 2, 2, **4**|
|`DENSE_RANK()`|같은 등수, 번호 연속|1, 2, 2, **3**|

```
ORDER BY 필수 — 없으면 에러
  ROW_NUMBER() OVER ()                   ← ❌ 에러
  ROW_NUMBER() OVER (ORDER BY 급여 DESC) ← ✅
```

```sql
-- Top-N 뽑기 핵심 패턴
-- ⚠️ 윈도우 함수는 WHERE 에 바로 못 씀 → CTE 로 감싸야 함
WITH ranked AS (
    SELECT 사원명, 급여,
        DENSE_RANK() OVER (ORDER BY 급여 DESC) AS rnk
    FROM 사원
)
SELECT * FROM ranked WHERE rnk <= 3;

-- 부서별 Top-N
WITH ranked AS (
    SELECT 부서, 사원명, 급여,
        DENSE_RANK() OVER (PARTITION BY 부서 ORDER BY 급여 DESC) AS rnk
    FROM 사원
)
SELECT * FROM ranked WHERE rnk <= 3;
```

## 언제 뭘 쓰나

|상황|함수|
|---|---|
|딱 N명만 (동점 무시)|`ROW_NUMBER()`|
|동점자 같은 등수, 번호 건너뜀|`RANK()`|
|동점자 같은 등수, 번호 연속|`DENSE_RANK()`|
|순위 번호 필요 없이 개수|`COUNT(*) + GROUP BY + HAVING`|

---

---

# ② ROW_NUMBER 특수 패턴

## 그룹별 최신 행 1건 추출 ⭐️

```sql
-- 모든 DB: ROW_NUMBER
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC) AS rn
    FROM Products
    WHERE change_date <= '2019-08-16'
)
SELECT * FROM ranked WHERE rn = 1;

-- PostgreSQL 전용: DISTINCT ON (더 간결)
SELECT DISTINCT ON (product_id)
       product_id, new_price
FROM Products
WHERE change_date <= '2019-08-16'
ORDER BY product_id, change_date DESC;
--       ↑ DISTINCT ON 컬럼이 ORDER BY 첫 번째여야 함
```

## Gaps and Islands — 연속 구간 찾기 ⭐️

```
연속된 값의 구간을 그룹으로 묶는 패턴
year - ROW_NUMBER() 가 같은 행끼리 = 같은 연속 구간
```

```sql
-- 패턴 1: 값 자체가 1씩 증가 (연도, 날짜)
WITH numbered AS (
    SELECT year,
           year - ROW_NUMBER() OVER (ORDER BY year) AS grp
    FROM awards
)
SELECT MIN(year) AS start, MAX(year) AS end, COUNT(*) AS cnt
FROM numbered
GROUP BY grp;

-- 패턴 2: 같은 값이 연속 등장 (숫자/문자)
WITH row_id AS (
    SELECT num,
           ROW_NUMBER() OVER (ORDER BY id)
           - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS grp
    FROM Logs
)
SELECT num FROM row_id GROUP BY num, grp HAVING COUNT(*) >= 3;

-- 패턴 3: 조건 필터 후 연속 id 그룹
WITH filtered AS (
    SELECT id, visit_date, people,
           id - ROW_NUMBER() OVER (ORDER BY id) AS grp
    FROM Stadium
    WHERE people >= 100   -- 조건 먼저 필터링
),
counted AS (
    SELECT *, COUNT(*) OVER (PARTITION BY grp) AS cnt
    FROM filtered
)
SELECT id, visit_date, people
FROM counted WHERE cnt >= 3 ORDER BY visit_date;
```

```
패턴 선택:
  값이 1씩 증가 (연도/날짜) → 패턴 1: year - rn
  같은 값 반복 등장          → 패턴 2: 전체rn - 값별rn
  조건 필터 후 연속 id        → 패턴 3: id - rn + COUNT OVER
```

---

---

# ③ LAG / LEAD — 이전/다음 행 ⭐️

```sql
LAG(컬럼)           -- 바로 윗줄 / NULL
LAG(컬럼, 7)        -- 7줄 위 (7일 전)
LAG(컬럼, 7, 0)     -- 7줄 위 / 없으면 0

LEAD(컬럼)          -- 바로 아랫줄
```

```sql
-- 전일 대비 증감
SELECT 날짜, 매출,
    LAG(매출) OVER (ORDER BY 날짜) AS 전일매출,
    매출 - LAG(매출) OVER (ORDER BY 날짜) AS 증감
FROM 매출;

-- 지역별로 따로 계산
LAG(매출) OVER (PARTITION BY 지역 ORDER BY 날짜)
--              ↑ 지역이 바뀌면 초기화
```

## 연속성 판단 — COUNT 가 아닌 LAG

```sql
-- ❌ COUNT 로 연속 판단 불가
HAVING COUNT(*) >= 2   -- "2번 이상" ≠ "연속"

-- ✅ LAG 로 간격 비교
SELECT 연도,
    연도 - LAG(연도) OVER (PARTITION BY 선수 ORDER BY 연도) AS 간격
FROM 기록;
-- 간격 = 4 → 올림픽 연속 출전
-- 간격 = 1 → 날짜/연도 연속
```

## 문제 키워드 → 함수

|키워드|함수|
|---|---|
|이전 / 전날 / 전월|`LAG`|
|다음 / 익일|`LEAD`|
|연속 / 연속 등장|`LAG` 간격 비교 or `ROW_NUMBER` 패턴|
|변화량 / 증감률|`LAG` + 뺄셈|

---

---

# ④ 집계 윈도우 함수

```
ORDER BY 없음 → 파티션 전체 합산 (모든 행에 같은 값)
ORDER BY 있음 → 현재 행까지 누적
```

```sql
SELECT 사원명, 급여,
    SUM(급여) OVER ()                       AS 전체합계,
    AVG(급여) OVER (PARTITION BY 부서)      AS 부서평균,
    SUM(급여) OVER (ORDER BY 입사일)        AS 누적합
FROM 사원;
```

## 이동 평균 ⭐️

```sql
-- 최근 3일 이동 평균 (오늘 포함)
AVG(매출) OVER (ORDER BY 날짜 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- 공식: N = 원하는 개수 - 1
-- 3일 이동평균: ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
-- 7일 이동평균: ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
```

## 누적합 주의 ⚠️

```sql
-- ❌ RANGE 기본값 → 같은 값이 한꺼번에 더해져 점프 발생
SUM(급여) OVER (ORDER BY 급여)

-- ✅ ROWS 명시 → 1줄씩 누적
SUM(급여) OVER (ORDER BY 급여 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

---

---

# ⑤ WHERE vs HAVING 날짜 범위 ⭐️

```sql
-- "기간 내에만 존재하는 것" 검증
HAVING MIN(sale_date) >= '2019-01-01'
   AND MAX(sale_date) <= '2019-03-31'

-- WHERE 로 하면 안 되는 이유:
-- WHERE BETWEEN → 기간 내 판매가 있는 것만 (기간 밖도 포함 가능)
-- HAVING MIN/MAX → 모든 데이터가 기간 안에 있다는 보장
```

---

---

# 초보자 실수 모음

|실수|해결|
|---|---|
|순위 함수에 `ORDER BY` 없이 `OVER()`|`OVER (ORDER BY 컬럼)` 필수|
|윈도우 함수를 `WHERE` 절에 직접 사용|CTE 감싸고 바깥에서 필터링|
|`LAST_VALUE()` 가 자기 자신만 출력|`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`|
|누적합이 점프함|`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 명시|
|COUNT 로 연속성 판단|`LAG` + 간격 비교 또는 ROW_NUMBER 패턴|
|WHERE 날짜 범위 → 일부 상품 사라짐|`HAVING MIN(날짜) >= 시작 AND MAX(날짜) <= 종료`|

---

----
