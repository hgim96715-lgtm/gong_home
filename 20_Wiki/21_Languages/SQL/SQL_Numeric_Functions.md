---
aliases:
  - 숫자함수
  - 반올림
  - 올림
  - 버림
  - 수학 함수
  - ROUND
  - TRUNC
  - SIGN
  - LEAST
  - GREATEST
tags:
  - SQL
related:
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_String_Functions]]"
  - "[[SQL_Type_Casting]]"
  - "[[SQL_Date_Functions]]"
---
# SQL_Numeric_Functions — 숫자 함수

## 한 줄 요약

```
지저분한 소수점을 다듬고
부호·나머지·범위를 자유자재로 다루는 수학 도구상자
```

---

---

# 전체 함수 한눈에 보기

|함수|용도|핵심|
|---|---|---|
|`ROUND(n, d)`|반올림|d 자릿수까지 남김|
|`TRUNC(n, d)`|버림(절사)|반올림 없이 잘라냄|
|`CEIL(n)`|무조건 올림|수직선 오른쪽 정수|
|`FLOOR(n)`|무조건 내림|수직선 왼쪽 정수|
|`ABS(n)`|절댓값|부호 제거|
|`SIGN(n)`|부호 판별|양수 1 / 0 / 음수 -1|
|`MOD(a, b)`|나머지|부호는 분자 따라감|
|`LEAST(a, b, ...)`|최솟값|**상한 제한**에 사용|
|`GREATEST(a, b, ...)`|최댓값|**하한 제한**에 사용|
|`LEAST(GREATEST(n, 0), 100)`|범위 클램핑|0~100 고정|

---

---

# ① ROUND — 반올림

```
ROUND(숫자, 자릿수)
  자릿수 이하에서 반올림
  자릿수 생략 = 0 (정수로)
  자릿수 음수 = 정수 부분 정리
```

```sql
SELECT ROUND(123.4567, 1);   -- 123.5   (둘째 자리에서 반올림)
SELECT ROUND(123.4567, 2);   -- 123.46  (셋째 자리에서 반올림)
SELECT ROUND(123.4567);      -- 123     (소수 첫째 자리에서 반올림)
SELECT ROUND(123.4567, 0);   -- 123     (위와 동일)

-- 음수 자릿수 → 정수 부분 정리
SELECT ROUND(123.4567, -1);  -- 120     (1의 자리에서 반올림)
SELECT ROUND(164.76,   -2);  -- 200     (10의 자리에서 반올림)
```

```
음수 자릿수 암기:
  -1 → 10 단위로  / -2 → 100 단위로  / -3 → 1000 단위로
  보고서에서 "만 원 단위", "천 단위" 절사할 때 씀
```

---

---

# ② TRUNC — 버림(절사)

```
TRUNC(숫자, 자릿수)
  반올림 없이 지정 자릿수 이하를 그냥 잘라냄
  금전 계산에서 "절사(버림)" 규칙에 사용

DB별 함수명:
  Oracle / PostgreSQL → TRUNC
  MySQL / MSSQL       → TRUNCATE
```

```sql
SELECT TRUNC(123.4567, 1);   -- 123.4  (5 뒤로 그냥 삭제)
SELECT TRUNC(123.4567, 2);   -- 123.45
SELECT TRUNC(123.4567, -1);  -- 120    (1의 자리 버림)

-- 금전 단위 절사
SELECT TRUNC(54321, -3);     -- 54000  (100원 단위 절사)
```

```
ROUND vs TRUNC:
  ROUND(123.5) = 124   (반올림 → 올라감)
  TRUNC(123.5) = 123   (그냥 잘라냄)

  금전 계산에서 "원 단위 절사" → 무조건 TRUNC
```

---

---

# ③ CEIL — 무조건 올림

```
CEIL(숫자)
  수직선 오른쪽(더 큰) 정수로 이동
  소수점이 0.1 이라도 있으면 다음 정수로

DB별 함수명:
  Oracle          → CEIL 만 지원
  MSSQL           → CEILING 만 지원
  PostgreSQL/MySQL → CEIL, CEILING 둘 다 지원
```

```sql
SELECT CEIL(123.1);    -- 124
SELECT CEIL(123.9);    -- 124
SELECT CEIL(123.0);    -- 123  (정수는 그대로)

-- 음수 주의 → 수직선 오른쪽 = 0에 가까운 쪽
SELECT CEIL(-22.3);    -- -22  (⚠️ -23 이 아님!)
```

```
음수 암기:
  CEIL(-22.3) = -22   수직선 오른쪽(더 큰 수)
  FLOOR(-22.3) = -23  수직선 왼쪽(더 작은 수)
```

---

---

# ④ FLOOR — 무조건 내림

```
FLOOR(숫자)
  수직선 왼쪽(더 작은) 정수로 이동
  용도: 만 나이 계산, 내림 처리
```

```sql
SELECT FLOOR(123.9);   -- 123
SELECT FLOOR(123.1);   -- 123

-- 음수 주의 → 수직선 왼쪽 = 더 작은 쪽
SELECT FLOOR(-22.3);   -- -23  (⚠️ -22 가 아님!)
```

```
TRUNC vs FLOOR 음수 비교:
  TRUNC(-22.3) = -22  (소수점 그냥 잘라냄, 방향 무관)
  FLOOR(-22.3) = -23  (수직선 왼쪽 → 더 작은 수)
```

---

---

# ⑤ ABS — 절댓값

```
ABS(숫자)
  부호를 떼고 무조건 양수
  용도: 두 값의 차이(격차) 자체를 구할 때
```

```sql
SELECT ABS(-10);    -- 10
SELECT ABS(15);     -- 15

-- 실전: 지연 시간 절댓값 (일찍 도착 = 음수 지연)
SELECT ABS(arr_delay) AS delay_minutes FROM train_delay;

-- 두 값의 차이
SELECT ABS(score_a - score_b) AS gap FROM results;
```

---

---

# ⑥ SIGN — 부호 판별

```
SIGN(숫자)
  양수 →  1
  0   →  0
  음수 → -1

용도: 증감 방향 라벨링 (성장/하락 분류)
```

```sql
SELECT SIGN(500);    -- 1
SELECT SIGN(0);      -- 0
SELECT SIGN(-25);    -- -1

-- 실전: 매출 증감 방향 분류
SELECT
    CASE SIGN(this_month - last_month)
        WHEN  1 THEN '성장'
        WHEN  0 THEN '유지'
        WHEN -1 THEN '하락'
    END AS trend
FROM monthly_sales;
```

---

---

# ⑦ MOD — 나머지

```
MOD(분자, 분모)
  나눗셈 나머지 반환
  PostgreSQL 은 % 연산자도 동일하게 작동

부호 규칙:
  나머지의 부호는 무조건 분자(앞 숫자) 부호를 따라감
  분모 부호는 결과에 영향 없음

분모가 0일 때:
  Oracle    → 에러 없이 분자 그대로 반환  (SQLD 시험 기준)
  PostgreSQL/MySQL → Division by zero 에러
```

```sql
SELECT MOD(10, 3);    -- 1
SELECT MOD(10, 0);    -- 10  (Oracle: 분자 반환 / PostgreSQL: 에러)

-- 부호 혼합
SELECT MOD(-10,  3);  -- -1  (분자 음수 → 결과 음수)
SELECT MOD( 10, -3);  --  1  (분자 양수 → 결과 양수)
SELECT MOD(-10, -3);  -- -1  (분자 음수 → 결과 음수)

-- PostgreSQL % 연산자
SELECT 10 % 3;        -- 1  (MOD 와 동일)
```

```
실전: A/B 테스트 버킷 분배
  customer_id 를 10으로 나눈 나머지로 그룹 균등 분배
```

```sql
SELECT
    customer_id,
    CASE WHEN MOD(customer_id, 2) = 0 THEN 'A' ELSE 'B' END AS bucket
FROM users;
```

---

---

# ⑧ LEAST / GREATEST — 범위 제어 ⭐️

```
LEAST(a, b, ...)    → 인자 중 가장 작은 값 반환  → 상한 제한에 사용
GREATEST(a, b, ...) → 인자 중 가장 큰 값 반환   → 하한 제한에 사용
```

```sql
SELECT LEAST(10, 5, 8);       -- 5
SELECT GREATEST(10, 5, 8);    -- 10

-- 상한 제한: 100을 넘지 못하게
SELECT LEAST(progress, 100);

-- 하한 제한: 0 미만으로 내려가지 못하게
SELECT GREATEST(stock - sold, 0);
```

## LEAST(GREATEST()) — 범위 클램핑

```
클램핑: 값이 [최솟값, 최댓값] 범위를 벗어나지 못하게 고정

LEAST(GREATEST(값, 최솟값), 최댓값)
  안쪽 GREATEST → 최솟값 이하로 내려가지 못하게 (하한)
  바깥 LEAST    → 최댓값 이상으로 올라가지 못하게 (상한)

읽는 순서: 안에서 밖으로
```

```sql
-- 진행률 0~100 고정
SELECT LEAST(GREATEST(progress_pct, 0), 100) AS pct;
--            ↑ 0 미만 방지              ↑ 100 초과 방지

-- 점수 0~100 범위 고정
SELECT LEAST(GREATEST(score, 0), 100) FROM exam_results;

-- 재고 음수 방지
SELECT GREATEST(stock - sold, 0) AS remaining FROM inventory;
```

>NULLIF 는 [[SQL_NULL_Functions]] 참조
>GREATEST,LEAST는 [[SQL_Numeric_Functions]] 참조 

---

---

# ⑨ PostgreSQL 정수 나눗셈 함정

```
정수 / 정수 = 정수 (소수점 그냥 버림)
파이썬 // 연산자와 동일한 동작
```

```sql
-- ❌ 소수점 증발
SELECT 10 / 3;          -- 3  (3.333... 이 아님!)

-- ✅ 하나를 실수로 만들어야 함
SELECT 10.0 / 3;        -- 3.3333...
SELECT 10 / 3::numeric; -- 3.3333...

-- 반올림까지 한번에
SELECT ROUND(10.0 / 3, 2);  -- 3.33

-- 실전: AVG 결과 반올림 (정수 입력이면 정수 반환될 수 있음)
SELECT ROUND(AVG(rating::numeric), 1) AS avg_score FROM reviews;
```

---

---

# 실무 패턴 모음

```sql
-- ① 금전 절사 (100원 단위 버림)
SELECT TRUNC(54321, -3);                     -- 54000

-- ② A/B 버킷 분배
SELECT CASE WHEN MOD(user_id, 2) = 0 THEN 'A' ELSE 'B' END FROM users;

-- ③ 페이지 수 계산 (올림)
SELECT CEIL(COUNT(*)::numeric / 10) AS pages FROM posts;

-- ④ 진행률 0~100 클램핑
SELECT LEAST(GREATEST(ROUND(elapsed / total * 100), 0), 100) AS pct;

-- ⑤ 증감 방향 라벨
SELECT CASE SIGN(this_week - last_week) WHEN 1 THEN '↑' WHEN -1 THEN '↓' ELSE '-' END;
```