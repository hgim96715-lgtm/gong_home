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


# SQL 윈도우 함수 (Window Functions)

## 개념 한 줄 요약

> **"`GROUP BY` 처럼 행을 압축하지 않고, 원본 행을 그대로 유지한 채로 계산 결과를 옆에 붙여주는 분석 함수."**

---

## 왜 필요한가?

문제: GROUP BY 부서 → 부서별 1줄만 남고 사원 개별 데이터 사라짐

해결: 윈도우 함수 → 개별 사원 데이터 유지 + 옆에 부서 평균 컬럼 추가


---

## 기본 문법 구조

```sql
함수명(인자) OVER (
    PARTITION BY 컬럼   -- 그룹 나누기 (선택)
    ORDER BY 컬럼       -- 정렬 기준   (선택 / 함수에 따라 필수)
    ROWS BETWEEN 시작 AND 끝  -- 범위 지정 (선택)
)
```

## PARTITION BY vs GROUP BY

```sql
-- GROUP BY: 그룹을 합쳐서 1줄로 압축 (원본 행 사라짐)
SELECT 부서, AVG(급여) FROM 사원 GROUP BY 부서;
-- → 부서별 1줄씩만 출력

-- PARTITION BY: 원본 행 유지하면서 계산값 추가
SELECT 사원명, 부서, 급여,
       AVG(급여) OVER (PARTITION BY 부서) AS 부서평균
FROM 사원;
-- → 원본 행 수 그대로 + 부서평균 컬럼 추가됨
```

```
GROUP BY   →  줄이는 것  (합쳐서 요약, 행 수 감소)
PARTITION BY →  붙이는 것 (원본 유지하면서 계산값 추가)
```

---

---

# ① 순위 함수

> **공동 순위 발생 시 그다음 번호를 어떻게 처리할지**에 따라 함수를 골라 쓴다.

## ⭐️ ORDER BY 필수

> **순위 함수는 반드시 `OVER (ORDER BY 컬럼)` 을 써야 한다.** `ORDER BY` 없이 `OVER ()` 만 쓰면 순위 기준이 없어서 에러가 발생한다.

```sql
-- ❌ 에러: 순위 기준이 없음
ROW_NUMBER() OVER ()

-- ✅ 정상: ORDER BY 필수
ROW_NUMBER() OVER (ORDER BY 급여 DESC)

-- ✅ PARTITION BY 와 함께 쓸 때도 ORDER BY 필수
DENSE_RANK() OVER (PARTITION BY 부서 ORDER BY 급여 DESC)
--                  ↑ 그룹 나누기 (선택)   ↑ 정렬 기준 (필수)
```

---

## 함수 한눈에 보기

|함수|동점자 처리|순위 번호|출력 예시|
|---|---|:-:|---|
|`ROW_NUMBER()`|동점 무시, 무조건 고유 번호|연속|1, 2, 3, 4|
|`RANK()`|동점자 같은 등수, 다음 번호 건너뜀|불연속|1, 2, 2, **4**|
|`DENSE_RANK()`|동점자 같은 등수, 번호 안 건너뜀|연속|1, 2, 2, **3**|

---

## ROW_NUMBER() — 무조건 고유한 번호

> "공동 순위 없이 **정확히 N명만** 뽑아줘" / 페이지네이션

```sql
ROW_NUMBER() OVER (ORDER BY 급여 DESC)
-- 같은 급여여도: 1, 2, 3, 4 (무조건 고유 번호)
```

## RANK() — 공동 순위 허용, 번호 건너뜀 (올림픽 메달 방식)

> 공동 2위가 2명이면 → 1, 2, 2, **4** (3등 없어짐)

```sql
RANK() OVER (ORDER BY 급여 DESC)
-- 동점자: 1, 2, 2, 4, 5 (3번 사라짐)
```

## DENSE_RANK() — 공동 순위 허용, 번호 연속 (밀집 순위)

> `<= 3` 으로 잘랐는데 **3위가 여러 명이면 전부 포함**하고 싶을 때

```sql
DENSE_RANK() OVER (ORDER BY 급여 DESC)
-- 동점자: 1, 2, 2, 3, 4 (번호 연속, 빈 자리 없음)
```

---

## 결과 비교표 (급여: 300, 200, 200, 100)

|급여|ROW_NUMBER()|RANK()|DENSE_RANK()|
|:-:|:-:|:-:|:-:|
|300|1|1|1|
|200|2|2|2|
|200|3|2|2|
|100|4|**4** (3 없음)|**3** (연속)|

---

## Top-N 뽑기 핵심 패턴

> **⚠️ 윈도우 함수는 `WHERE` 절에서 바로 쓸 수 없다!** CTE(WITH절) 또는 인라인 뷰로 감싼 뒤 바깥에서 필터링해야 한다.

```sql
WITH RankedEmp AS (
    SELECT
        사원명, 급여,
        ROW_NUMBER() OVER (ORDER BY 급여 DESC) AS rn  -- 딱 3명
        -- RANK()       OVER (ORDER BY 급여 DESC) AS rn  -- 공동 2등 있으면 3명
        -- DENSE_RANK() OVER (ORDER BY 급여 DESC) AS rn  -- 3위까지 동점자 전원
    FROM 사원
)
SELECT 사원명, 급여, rn AS 순위
FROM RankedEmp
WHERE rn <= 3;  -- 바깥 쿼리에서 필터링!
```

## 부서별 Top-N — PARTITION BY 활용

```sql
WITH RankedEmp AS (
    SELECT
        부서, 사원명, 급여,
        DENSE_RANK() OVER (
            PARTITION BY 부서    -- 부서가 바뀔 때마다 순위 1부터 리셋
            ORDER BY 급여 DESC
        ) AS rn
    FROM 사원
)
SELECT 부서, 사원명, 급여, rn AS 순위
FROM RankedEmp
WHERE rn <= 3;
```

---

## 언제 뭘 써야 하나?

|상황|추천 함수|이유|
|---|---|---|
|무조건 1명만 (중복 허용 X)|`ROW_NUMBER()`|동점이어도 1명만|
|동점자 같은 등수 인정|`RANK()`|공동 2위 인정|
|등수 번호가 연속이어야 할 때|`DENSE_RANK()`|빈 등수 없이 연속|
|상위 N명 + 동점자 전원 포함|`RANK()`|공동 3위 2명이면 둘 다|
|페이지네이션 / 행 번호|`ROW_NUMBER()`|고유 번호 필요|

## RANK vs COUNT

```
"순위 번호(1등, 2등...)" 가 결과에 필요한가?
        ↓ YES                    ↓ NO
     RANK 써라         COUNT(*) + GROUP BY + HAVING
```

|문제에 이 단어가 보이면|꺼낼 도구|
|---|---|
|"1위", "상위 N위", "순위"|`RANK()` / `DENSE_RANK()`|
|"횟수", "몇 번", "건수"|`COUNT(*)`|
|"중복 없이 한 명만"|`ROW_NUMBER()`|
|"각 그룹에서 최고값"|`RANK()` + `WHERE rnk = 1`|

---

---

# ② 집계 함수 (윈도우 버전)

> `SUM`, `AVG`, `MAX`, `MIN`, `COUNT` 를 **원본 행 유지 상태**로 계산한다. `ORDER BY` 유무에 따라 **전체 합산** vs **누적 합산** 이 된다.

```
ORDER BY 없음  →  파티션 전체 합산 (같은 파티션의 모든 행에 동일한 값 복사)
ORDER BY 있음  →  첫 행부터 현재 행까지 누적
```

```sql
SELECT
    부서, 사원명, 입사일자, 주문금액, 급여,

    -- 전체 합산 (ORDER BY 없음) → 모든 행에 동일한 총합
    SUM(주문금액) OVER() AS 총_주문금액,

    -- 파티션 평균 (PARTITION BY만, ORDER BY 없음)
    ROUND(AVG(급여) OVER (PARTITION BY 부서), 2) AS 부서평균급여,

    -- 파티션 최댓값
    MAX(급여) OVER (PARTITION BY 부서) AS 부서내_최고급여,

    -- 누적 합산 (ORDER BY 있음) → 입사일 순으로 한 줄씩 누적
    SUM(주문금액) OVER (ORDER BY 입사일자) AS 누적_주문금액,

    -- 누적 최솟값 → 날짜가 지날 때마다 역대 최저 갱신
    MIN(급여) OVER (ORDER BY 입사일자) AS 지금까지_최저급여

FROM 사원테이블;
```

## 결과 이해 예시 (영업부 4명)

|사원명|입사일|급여|MAX OVER(PARTITION BY)|MIN OVER(ORDER BY)|계산 원리|
|---|---|---|:-:|:-:|---|
|김신입|1/1|300|**500**|**300**|MAX: 영업부 전체 최고=500 / MIN: 첫 행=300|
|이대리|1/2|500|**500**|**300**|MAX: 동일 / MIN: 최저 여전히 300|
|박과장|1/3|400|**500**|**300**|MAX: 동일 / MIN: 최저 유지|
|최인턴|1/4|200|**500**|**200 🔥**|MAX: 동일 / MIN: 더 낮은 값 등장 → 갱신|

---

---

# ③ 행 순서 함수 (SQL Server 미지원)

> **현재 행을 기준으로 이전/다음 행의 값을 끌어온다.** 시계열 데이터(날짜별 변화량) 분석의 핵심 도구.

|함수|역할|
|---|---|
|`LAG()`|이전 행 값 가져오기|
|`LEAD()`|다음 행 값 가져오기|
|`FIRST_VALUE()`|파티션 내 첫 번째 값|
|`LAST_VALUE()`|파티션 내 마지막 값 ⚠️ 범위 지정 필수|

---

## LAG() — 이전 행 가져오기

```sql
-- 문법: LAG(컬럼, 건너뛸_행_수, NULL_대체값)
-- 건너뛸_행_수 생략 → 기본값 1 (바로 윗줄)

-- 기본: 바로 어제 값
SELECT 접속일자, 접속자수,
    LAG(접속자수) OVER (ORDER BY 접속일자) AS 전일_접속자수
FROM 트래픽로그;

-- 심화: 7일 전 값, 없으면 0으로 대체
SELECT 접속일자, 접속자수,
    LAG(접속자수, 7, 0) OVER (ORDER BY 접속일자) AS 일주일전_접속자수
FROM 트래픽로그;
```

## LEAD() — 다음 행 가져오기

```sql
-- LAG 와 구조 동일, 방향만 반대
SELECT 접속일자, 접속자수,
    LEAD(접속자수) OVER (ORDER BY 접속일자) AS 익일_접속자수
FROM 트래픽로그;
```

## FIRST_VALUE() — 파티션 내 첫 번째 값

```sql
SELECT 접속일자, 접속자수,
    FIRST_VALUE(접속자수) OVER (PARTITION BY 월 ORDER BY 접속일자) AS 월초_접속자수
FROM 트래픽로그;
```

## LAST_VALUE() — 파티션 내 마지막 값 ⚠️

> 윈도우 기본 범위는 "현재 행까지". 그냥 쓰면 자기 자신을 마지막 값으로 출력한다. **`UNBOUNDED FOLLOWING` 으로 범위를 끝까지 열어야 한다.**

```sql
SELECT 접속일자, 접속자수,
    LAST_VALUE(접속자수) OVER (
        PARTITION BY 월
        ORDER BY 접속일자
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- ← 필수!
    ) AS 월말_접속자수
FROM 트래픽로그;
```

---

## 연속성 문제 — COUNT 가 아니라 LAG 로 판단한다

```
두 번 이상 참가 ≠ 연속 참가
연속성 = "개수"의 문제가 아니라 "시간 간격"의 문제
```

```sql
-- ❌ 잘못된 접근
HAVING COUNT(*) >= 2

-- ✅ 올바른 접근: LAG 로 간격 비교
SELECT 선수명, 참가연도,
    LAG(참가연도) OVER (PARTITION BY 선수명 ORDER BY 참가연도) AS 이전_연도,
    참가연도 - LAG(참가연도) OVER (PARTITION BY 선수명 ORDER BY 참가연도) AS 간격
FROM 올림픽참가기록;
-- 간격 = 4 이면 올림픽 연속 출전
```

|상황|공식|기댓값|
|---|---|---|
|올림픽 연속 출전|`현재연도 - LAG(연도) = 4`|4|
|날짜 연속|`현재날짜 - LAG(날짜) = 1`|1|
|월 연속|`현재월 - LAG(월) = 1`|1|

## 문제 키워드 신호

|키워드|필요한 함수|
|---|---|
|**연속** (연속 출전, 연속 달성)|`LAG` + 간격 비교|
|**이전** (이전 달 대비, 전날 대비)|`LAG`|
|**다음** (다음 달 예측, 익일 비교)|`LEAD`|
|**변화** (증가율, 변화량)|`LAG` + 뺄셈|
|**증가** (연속 증가, 꾸준히 상승)|`LAG` + 조건 비교|

---

---

# ④ 비율 함수 (SQL Server 미지원)

| 함수                  |     결과값 범위      | 핵심 역할              |
| ------------------- | :-------------: | ------------------ |
| `RATIO_TO_REPORT()` |      0 ~ 1      | 전체 합 대비 내 비율       |
| `PERCENT_RANK()`    | 0.0 ~ 1.0 (0부터) | 상대적 순위 비율 (상위 몇 %) |
| `CUME_DIST()`       |  (0, 1] (0 불가)  | 누적 분포 비율 (나 포함)    |
| `NTILE(N)`          |   1 ~ N (정수)    | N개 등급으로 균등 분할      |

---

## RATIO_TO_REPORT() — 전체 합 대비 내 비율

> Oracle 전용. PostgreSQL 은 직접 나눗셈으로 대체. `OVER()` 안에 `ORDER BY` 쓰지 않는다.

```sql
SELECT 사원명, 급여,
    RATIO_TO_REPORT(급여) OVER() AS 오라클_급여비율,         -- Oracle 전용
    ROUND(급여 / SUM(급여) OVER(), 2) AS 포그레_급여비율     -- PostgreSQL 대체
FROM 사원;
```

## PERCENT_RANK() — 상대적 위치를 비율로

> 공식: `(현재 순위 - 1) / (전체 행 수 - 1)` 1등 = 0.0, 꼴찌 = 1.0

```sql
-- "상위 30% 이내 직원만 추출"
WITH RankedEmp AS (
    SELECT 사원명, 급여,
        PERCENT_RANK() OVER (ORDER BY 급여 DESC) AS pct_rank
    FROM 사원
)
SELECT 사원명, 급여
FROM RankedEmp
WHERE pct_rank <= 0.3;
```

## CUME_DIST() — 누적 분포 비율

> 나보다 작거나 같은 값이 전체의 몇 % 인지 (나 자신 포함) 결과가 절대 0 이 될 수 없음

```sql
SELECT 사원명, 급여,
    CUME_DIST() OVER (ORDER BY 급여 DESC) AS 누적_분포비율
FROM 사원;
```

## NTILE(N) — N개 등급으로 균등 분할

```sql
SELECT 사원명, 급여,
    NTILE(4) OVER (ORDER BY 급여 DESC) AS 급여_4분위
FROM 사원;
```

**⚠️ 주의사항 2가지:**

```
① 나머지 우선 할당: 행 수가 N으로 안 떨어지면 맨 위 그룹부터 1개씩 더 배정
② 동점자 강제 분리: 같은 값이어도 경계선에 걸리면 다른 그룹으로 강제 분리
```

## 비율 함수 비교표

|함수|범위|1등 값|꼴찌 값|핵심|
|---|:-:|:-:|:-:|---|
|`RATIO_TO_REPORT()`|0 ~ 1|-|-|파티션 내 합 = 1|
|`PERCENT_RANK()`|0.0 ~ 1.0|**0.0**|1.0|1등 무조건 0|
|`CUME_DIST()`|(0, 1]|0 불가|**1.0**|0 이 절대 안 됨|
|`NTILE(N)`|1 ~ N|1|N|결과는 정수|

---

---

# ⑤ Windowing 절 — 범위 세밀 지정

> `ORDER BY` 로 줄을 세웠다면, 현재 행 기준으로 어디서 어디까지 연산할지 지정한다. 이동 평균(최근 3일 평균 매출) 구할 때 필수.

## 키워드 해석

|키워드|해석|
|---|---|
|`UNBOUNDED PRECEDING`|파티션의 **맨 첫 행**|
|`UNBOUNDED FOLLOWING`|파티션의 **맨 끝 행**|
|`CURRENT ROW`|**현재 행**|
|`1 PRECEDING`|**바로 1줄 앞**|
|`1 FOLLOWING`|**바로 1줄 뒤**|

## ROWS vs RANGE

|구분|기준|특징|
|---|---|---|
|`ROWS`|물리적 줄 수|값이 같든 다르든 진짜 줄 수로 셈|
|`RANGE`|논리적 값|같은 값(동점자)을 하나의 덩어리로 묶어서 셈|

```sql
SELECT 날짜, 매출,
    -- 이동 평균 (앞 1줄 ~ 뒤 1줄, 총 3줄)
    AVG(매출) OVER (
        ORDER BY 날짜
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS 이동평균_3일,

    -- 완벽한 누적합
    SUM(매출) OVER (
        ORDER BY 날짜
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS 누적합
FROM 매출테이블;
```

---

## ⚠️ 기본값의 배신 — 누적합이 점프하는 이유

> `ORDER BY` 만 쓰고 범위 생략 시 → 암묵적으로 **`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`** 발동 같은 정렬 기준값(급여, 날짜)을 가진 행들이 덩어리로 묶여 한 번에 더해져 숫자가 껑충 점프한다.

```sql
SELECT 사원명, 급여,
    -- 기본값 (RANGE) → 같은 급여 묶어서 한 번에 더해버림
    SUM(급여) OVER (ORDER BY 급여) AS 누적합_기본,

    -- 해결책 (ROWS) → 1줄씩 차곡차곡
    SUM(급여) OVER (ORDER BY 급여 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS 누적합_정확
FROM 사원;
```

|사원명|급여|누적합 (RANGE 기본)|누적합 (ROWS 해결)|
|---|---|:-:|:-:|
|박신입|100|100|100|
|**나보통**|**200**|**500 💥**|**300**|
|**김대리**|**200**|**500**|**500**|
|최과장|300|800|800|

---

---

# 초보자 실수 모음

|실수|해결|
|---|---|
|순위 함수에 `ORDER BY` 없이 `OVER()` 만 사용|`OVER (ORDER BY 컬럼)` 필수|
|윈도우 함수를 `WHERE` 절에 사용|CTE 또는 인라인 뷰로 감싸고 바깥에서 필터링|
|`LAST_VALUE()` 가 자기 자신을 출력|`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` 추가|
|`SUM() OVER(ORDER BY)` 누적합이 점프|`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 명시|
|COUNT 로 연속성 판단|`LAG` + 간격 비교로 대체|

---