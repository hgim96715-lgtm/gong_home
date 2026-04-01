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

```
문제: GROUP BY 부서 → 부서별 1줄만 남고 사원 개별 데이터 사라짐
해결: 윈도우 함수 → 개별 사원 데이터 유지 + 옆에 부서 평균 컬럼 추가
```

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
GROUP BY     줄이는 것  (합쳐서 요약, 행 수 감소)
PARTITION BY 붙이는 것  (원본 유지하면서 계산값 추가)
```

## 실행 순서 — Window 함수는 GROUP BY 이후 ⭐️

```
SQL 실행 순서:
  FROM → WHERE → GROUP BY → HAVING → SELECT(Window 함수) → ORDER BY

Window 함수는 SELECT 계산 중에 실행
→ GROUP BY / HAVING 보다 나중에 실행
→ GROUP BY 로 이미 행이 합쳐진 결과 위에서 동작
```

```
⚠️ GROUP BY + Window 함수 동시 사용 불가:

  GROUP BY  → 행을 합쳐버림 (여러 행 → 1행)
  Window 함수 → 각 행을 유지하면서 계산

  둘은 방향이 반대 → 함께 쓰면 충돌
```

```sql
-- ❌ 에러 — raw 데이터에 GROUP BY + Window 함수 동시
SELECT
    region,
    SUM(hvec) OVER (PARTITION BY region) AS region_total
FROM er_realtime
GROUP BY region;
-- ERROR: column "er_realtime.hvec" must appear in GROUP BY clause

-- ✅ 해결 1 — GROUP BY 빼고 Window 함수만
SELECT
    hpid, region, hvec,
    SUM(hvec) OVER (PARTITION BY region) AS region_total
FROM er_realtime;

-- ✅ 해결 2 — GROUP BY 결과 위에 Window 함수 적용 (서브쿼리)
SELECT
    region,
    SUM(hvec) AS total,
    SUM(SUM(hvec)) OVER () AS grand_total   -- GROUP BY 결과에 Window 함수
FROM er_realtime
GROUP BY region;
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

```
ROW_NUMBER() 의 또 다른 활용 — 그룹별 최신 행 1건 추출

PARTITION BY 와 ORDER BY 를 조합하면
"각 그룹에서 가장 최신(또는 최고/최저) 행 1건만" 뽑을 수 있음
→ PostgreSQL 에서는 DISTINCT ON 이 같은 역할을 더 간결하게 함
→ MySQL / Oracle / SQL Server 등 다른 DB 에서는 ROW_NUMBER() 가 표준 대체 방법
```

```sql
-- 열차별 가장 최신 상태 1건만 (MySQL / Oracle / SQL Server 공통)
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY trn_no ORDER BY created_at DESC) AS rn
    FROM train_realtime
) t
WHERE rn = 1;

-- PostgreSQL 에서는 DISTINCT ON 으로 더 간결하게
-- → [[SQL_DISTINCT_vs_GROUP_BY]] 참고
```

## ROW_NUMBER() — Gaps and Islands (연속 구간 찾기) ⭐️

```
"연속된 년도에 수상한 기간은 몇 년인가?"
"연속으로 출석한 구간은?"

이런 문제를 Gaps and Islands(간격과 섬) 문제라고 부름
  섬(Island)   = 연속된 데이터 구간
  간격(Gap)    = 끊긴 구간

ROW_NUMBER() 를 쓰면 순위를 매기는 게 아니라
연속 구간을 그룹으로 묶는 데 활용할 수 있음
```

## 왜 year - ROW_NUMBER() 가 연속 구간이 되는가

```
핵심 원리:
  연속된 년도는 1씩 증가 → year - ROW_NUMBER() 는 항상 같은 값
  끊기면 year 가 점프 → year - ROW_NUMBER() 가 달라짐

예시 (수상 년도: 2010, 2011, 2012, 2015, 2016):

  year  ROW_NUMBER()  year - ROW_NUMBER()
  2010       1             2009  ← 같은 값 → 같은 그룹
  2011       2             2009  ←
  2012       3             2009  ←
  2015       4             2011  ← 다른 값 → 다른 그룹 (2013/2014 빠짐)
  2016       5             2011  ←

year - ROW_NUMBER() 가 같은 행끼리 = 연속된 구간
```

```sql
-- 연도별 연속 수상 구간 찾기
WITH numbered AS (
    SELECT
        year,
        ROW_NUMBER() OVER (ORDER BY year) AS rn,
        year - ROW_NUMBER() OVER (ORDER BY year) AS grp   -- 연속 그룹 키
    FROM awards
),
grouped AS (
    SELECT
        grp,
        MIN(year) AS start_year,
        MAX(year) AS end_year,
        COUNT(*)  AS consecutive_years   -- 연속 기간
    FROM numbered
    GROUP BY grp
)
SELECT
    start_year,
    end_year,
    consecutive_years
FROM grouped
ORDER BY start_year;

-- 결과:
-- start_year  end_year  consecutive_years
-- 2010        2012      3   ← 3년 연속
-- 2015        2016      2   ← 2년 연속
```

```
단계별 이해:

1단계 (numbered):
  각 행에 ROW_NUMBER 부여 + year - rn 으로 그룹 키 생성

2단계 (grouped):
  같은 grp 끼리 묶어서
  시작년도 / 끝년도 / 기간 계산

3단계:
  연속 구간별 결과 출력
```

```sql
-- 응용: 선수별 연속 출전 구간
WITH numbered AS (
    SELECT
        player_name,
        year,
        ROW_NUMBER() OVER (PARTITION BY player_name ORDER BY year) AS rn,
        year - ROW_NUMBER() OVER (PARTITION BY player_name ORDER BY year) AS grp
    FROM olympic_records
)
SELECT
    player_name,
    MIN(year) AS start_year,
    MAX(year) AS end_year,
    COUNT(*) AS consecutive_years
FROM numbered
GROUP BY player_name, grp
HAVING COUNT(*) >= 2   -- 2회 이상 연속만
ORDER BY player_name, start_year;
```

```
PARTITION BY 추가 → 선수별로 각각 그룹 계산
HAVING COUNT(*) >= 2 → 1회짜리 단독 수상 제외

LAG 와 비교:
  LAG     → "연속인지 아닌지" 판단 (간격 비교)
  year-rn → "연속 구간을 하나의 그룹으로 묶기"

  LAG 로 연속 여부 판단 → 그룹별 기간 집계 필요
  year-rn 패턴 → 한 번에 그룹화 + 집계 가능
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

### LAG() 괄호 안에 뭘 넣나 ⭐️

```
LAG(① 컬럼, ② 몇 칸 전, ③ NULL 대체값)

①  가져올 값         필수
②  몇 행 이전        생략 가능 (기본값 1 = 바로 윗줄)
③  없으면 뭘로 채울지  생략 가능 (기본값 NULL)
```


```sql
LAG(접속자수)           -- 바로 윗줄 값 / 없으면 NULL
LAG(접속자수, 1)        -- 위와 동일 (1 생략 가능)
LAG(접속자수, 7)        -- 7줄 위 값 (7일 전) / 없으면 NULL
LAG(접속자수, 7, 0)     -- 7줄 위 값 / 없으면 0
LAG(접속자수, 1, -1)    -- 바로 윗줄 값 / 없으면 -1
```

```
③ 언제 NULL 이 나오나:
  첫 번째 행 → 이전 행이 없음 → NULL (또는 대체값)
  7일 전 조회인데 데이터가 5일치뿐 → 처음 7행은 NULL

  0 vs NULL:
  0 넣으면 → SUM/AVG 계산에 포함됨
  NULL 이면 → SUM/AVG 계산에서 제외됨
  → 상황에 맞게 선택
```

```sql
-- 문법 정리
-- LAG(컬럼, 건너뛸_행_수, NULL_대체값)
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

---

## OVER() 안에 PARTITION BY 뭘 기준으로 하나 ⭐️

```
PARTITION BY = "이 단위로 쪼개서 각각 따로 계산해줘"

없으면   → 전체 테이블을 하나로 보고 계산
있으면   → 그룹별로 쪼개서 각 그룹 안에서 계산
```

```
판단 기준:
  "이 계산이 전체 기준인가, 그룹 기준인가?"

  전체 기준  → PARTITION BY 없이 OVER (ORDER BY ...)
  그룹 기준  → OVER (PARTITION BY 그룹기준 ORDER BY ...)
```

```sql
-- 예시 데이터
-- 날짜        지역   접속자수
-- 2024-01-01  서울   100
-- 2024-01-02  서울   120
-- 2024-01-01  부산   80
-- 2024-01-02  부산   90

-- PARTITION BY 없음 → 전체 테이블 기준으로 이전 행
SELECT 날짜, 지역, 접속자수,
    LAG(접속자수) OVER (ORDER BY 날짜) AS 이전값
FROM 트래픽로그;
-- 날짜        지역  접속자수  이전값
-- 2024-01-01  서울  100      NULL  ← 첫 행
-- 2024-01-01  부산  80       100   ← 서울 다음 행
-- 2024-01-02  서울  120      80    ← 부산 다음 행 (지역 뒤섞임!)
-- 2024-01-02  부산  90       120

-- PARTITION BY 지역 → 지역별로 쪼개서 각자 계산
SELECT 날짜, 지역, 접속자수,
    LAG(접속자수) OVER (PARTITION BY 지역 ORDER BY 날짜) AS 이전값
FROM 트래픽로그;
-- 날짜        지역  접속자수  이전값
-- 2024-01-01  부산  80       NULL  ← 부산 첫 행
-- 2024-01-02  부산  90       80    ← 부산 이전값
-- 2024-01-01  서울  100      NULL  ← 서울 첫 행
-- 2024-01-02  서울  120      100   ← 서울 이전값 ✅
```

```
PARTITION BY 선택 기준 정리:

  전체 추이 분석
  → PARTITION BY 없음
  → "모든 데이터에서 바로 이전 행"

  그룹별 추이 분석
  → PARTITION BY 그룹컬럼
  → "부서별 / 지역별 / 선수별 각자 이전 행"

  실전 예시:
    선수별 연속 출전  → PARTITION BY 선수명
    지역별 전일 접속  → PARTITION BY 지역
    전체 일별 추이   → PARTITION BY 없음
```

```sql
-- 실전: 선수별 연속 출전 확인
LAG(참가연도) OVER (PARTITION BY 선수명 ORDER BY 참가연도)
--                  ↑ 선수가 바뀌면 리셋    ↑ 연도순 정렬

-- 실전: 전체 일별 전일 대비 증감
LAG(매출) OVER (ORDER BY 날짜)
--              PARTITION BY 없음 → 전체 흐름
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

|함수|결과값 범위|핵심 역할|
|---|:-:|---|
|`RATIO_TO_REPORT()`|0 ~ 1|전체 합 대비 내 비율|
|`PERCENT_RANK()`|0.0 ~ 1.0 (0부터)|상대적 순위 비율 (상위 몇 %)|
|`CUME_DIST()`|(0, 1] (0 불가)|누적 분포 비율 (나 포함)|
|`NTILE(N)`|1 ~ N (정수)|N개 등급으로 균등 분할|

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

```
ORDER BY 로 줄을 세웠다면
현재 행 기준으로 어디서 어디까지 연산할지 지정
이동 평균 / 누적합 구할 때 필수
```

## 키워드 해석

|키워드|해석|
|---|---|
|`UNBOUNDED PRECEDING`|파티션의 맨 첫 행|
|`UNBOUNDED FOLLOWING`|파티션의 맨 끝 행|
|`CURRENT ROW`|현재 행|
|`N PRECEDING`|N줄 앞|
|`N FOLLOWING`|N줄 뒤|

## ROWS vs RANGE

|구분|기준|특징|
|---|---|---|
|`ROWS`|물리적 줄 수|값이 같든 다르든 진짜 줄 수로 셈|
|`RANGE`|논리적 값 범위|ORDER BY 컬럼 값이 같은 행들을 덩어리로 묶어서 셈|

---

## 이동 평균 — 공식 패턴 ⭐️

```
AVG(컬럼) OVER (
    ORDER BY 기준
    ROWS BETWEEN N PRECEDING AND CURRENT ROW
)

→ 현재 행 포함 + 과거 N개 = 총 N+1개 평균
→ 거의 공식처럼 외워도 됨
```

```sql
SELECT 날짜, 매출,
    -- 최근 3일 이동 평균 (오늘 포함 3개)
    AVG(매출) OVER (
        ORDER BY 날짜
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS 이동평균_3일,

    -- 최근 7일 이동 평균 (오늘 포함 7개)
    AVG(매출) OVER (
        ORDER BY 날짜
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS 이동평균_7일,

    -- 앞뒤 1줄 포함 3줄 평균 (중앙 기준)
    AVG(매출) OVER (
        ORDER BY 날짜
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS 이동평균_중앙
FROM 매출테이블;
```

```
ROWS BETWEEN N PRECEDING AND CURRENT ROW 읽는 법:
  N PRECEDING  = 과거 N개
  CURRENT ROW  = 현재 행
  → 현재 포함 + 과거 N개 = 총 N+1개

  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  → 오늘 포함 3개 (그제, 어제, 오늘)
  → "최근 3일 이동 평균"
```

## 시간 단위 → 행 개수 변환 ⭐️

```
시계열 데이터는 "시간"을 "행 개수"로 바꿔서 생각해야 함

예시:
  10분 단위 데이터 (10분마다 1행)
  → 1시간 = 6행
  → "1시간 이동 평균" = ROWS BETWEEN 5 PRECEDING AND CURRENT ROW

  5분 단위 데이터
  → 1시간 = 12행
  → "1시간 이동 평균" = ROWS BETWEEN 11 PRECEDING AND CURRENT ROW

계산 공식:
  필요한 N = (원하는 시간 / 데이터 간격) - 1
  1시간 / 10분 - 1 = 6 - 1 = 5 → ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
```

```sql
-- 실전: 10분 단위 응급실 병상 데이터로 1시간 이동 평균
SELECT
    measured_at,
    hvec,
    AVG(hvec) OVER (
        ORDER BY measured_at
        ROWS BETWEEN 5 PRECEDING AND CURRENT ROW   -- 10분 × 6 = 1시간
    ) AS moving_avg_1h
FROM er_realtime
ORDER BY measured_at;
```

## end_at 보정 패턴 — INTERVAL ⭐️

```
로그/시계열 데이터에서 자주 나오는 패턴
데이터가 시작 시각(start_at)만 있고 끝 시각이 없을 때
→ 측정 주기를 더해서 end_at 계산
```

```sql
-- 10분 단위 로그 → end_at = measured_at + 10분
SELECT
    measured_at,
    measured_at + INTERVAL '10 minute' AS end_at,
    hvec
FROM er_realtime;

-- 1시간 단위 집계 데이터
SELECT
    stat_hour,
    stat_hour + INTERVAL '1 hour' AS end_at,
    avg_beds
FROM er_hourly_stats;
```

```
활용:
  Superset 타임라인 차트에서 시작~끝 범위 지정
  JOIN 시 시간 구간 조건
  WHERE measured_at BETWEEN start_at AND start_at + INTERVAL '10 minute'
```

## 누적합 — 완벽한 패턴

```sql
SELECT 날짜, 매출,
    -- 완벽한 누적합 (ROWS 명시)
    SUM(매출) OVER (
        ORDER BY 날짜
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS 누적합
FROM 매출테이블;
```

---

## ⚠️ 기본값의 배신 — 누적합이 점프하는 이유

```
ORDER BY 만 쓰고 범위 생략 시
→ 암묵적으로 RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW 발동
→ ORDER BY 기준 컬럼 값이 같은 행들이 덩어리로 묶여 한 번에 더해짐
→ 숫자가 껑충 점프
```

```sql
SELECT 사원명, 급여,
    -- 기본값 (RANGE) → 같은 급여 묶어서 한 번에 더해버림
    SUM(급여) OVER (ORDER BY 급여) AS 누적합_기본,

    -- 해결책 (ROWS) → 1줄씩 차곡차곡
    SUM(급여) OVER (
        ORDER BY 급여
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS 누적합_정확
FROM 사원;
```

|사원명|급여|누적합 (RANGE 기본)|누적합 (ROWS 해결)|
|---|---|:-:|:-:|
|박신입|100|100|100|
|나보통|200|500 💥|300|
|김대리|200|500|500|
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

## 관련 노트

- [[SQL_Aggregate_GROUP_BY]] — COUNT · SUM · AVG · MAX · MIN 기본
- [[SQL_Top_N_Query]] — ROWNUM · FETCH FIRST
- [[SQL_CASE_WHEN]] — 조건부 집계

---

---

# ⑥ MAX + JOIN 패턴 — 동점까지 모두 가져오기

## 언제 필요한가

```
"각 그룹에서 최댓값인 행을 가져와라"
→ 단순히 MAX 값만 구하면 끝이 아님
→ 그 값을 가진 원본 행의 다른 컬럼들도 함께 필요

예시:
  개발사별 가장 많이 팔린 플랫폼은?
  → 개발사 + 플랫폼 + 판매량 다 필요
  → 동점 플랫폼이 있으면 둘 다 가져와야 함

Window 함수 RANK() 로도 가능하지만
동점(공동 1위) 처리가 중요한 상황에서
MAX + JOIN 패턴이 더 직관적
```

## 기본 흐름

```
1단계: GROUP BY 로 집계 (원하는 단위로 합산)
2단계: MAX() 로 그룹별 최댓값 구하기
3단계: 1단계 결과와 JOIN → 최댓값을 가진 행만 남김
```

```sql
-- 실전: 개발사별 가장 많이 팔린 플랫폼 (동점 포함)

-- 1단계: 개발사 + 플랫폼 별 총 판매량 집계
WITH sum_game AS (
    SELECT
        c.name AS developer,
        p.name AS platform,
        SUM(sales_na + sales_eu + sales_jp + sales_other) AS sales
    FROM games g
    JOIN companies c ON g.developer_id = c.company_id
    JOIN platforms p ON g.platform_id  = p.platform_id
    GROUP BY c.name, p.name
),

-- 2단계: 개발사별 최대 판매량
max_sales AS (
    SELECT
        developer,
        MAX(sales) AS max_sale
    FROM sum_game
    GROUP BY developer
)

-- 3단계: 집계 결과 + 최댓값 JOIN → 최댓값 행만 남김
SELECT
    s.developer,
    s.platform,
    s.sales
FROM sum_game s
JOIN max_sales ms
    ON s.developer = ms.developer
   AND s.sales     = ms.max_sale   -- ← 최댓값과 일치하는 행만 남김
;
```

## 왜 JOIN 이 필요한가?

```sql
-- ❌ 이렇게 하면 안 됨
SELECT developer, platform, MAX(sales)
FROM sum_game
GROUP BY developer;

-- → platform 컬럼을 GROUP BY 에 넣으면
--   developer + platform 조합으로 집계 → 개발사별 최대가 아님
-- → platform 을 GROUP BY 에서 빼면
--   SELECT 에 platform 을 쓸 수 없음 (집계되지 않은 컬럼)
```

```
문제의 핵심:
  MAX(sales) 를 가진 행이 어느 platform 인지 알려면
  MAX 값을 구한 뒤 원본과 다시 연결해야 함
  → 이게 MAX + JOIN 패턴
```

## RANK() 로 대체 가능

```sql
-- RANK() 버전 (동점 포함 처리 동일)
WITH sum_game AS (
    SELECT
        c.name AS developer,
        p.name AS platform,
        SUM(sales_na + sales_eu + sales_jp + sales_other) AS sales
    FROM games g
    JOIN companies c ON g.developer_id = c.company_id
    JOIN platforms p ON g.platform_id  = p.platform_id
    GROUP BY c.name, p.name
),
ranked AS (
    SELECT
        developer,
        platform,
        sales,
        RANK() OVER (PARTITION BY developer ORDER BY sales DESC) AS rnk
    FROM sum_game
)
SELECT developer, platform, sales
FROM ranked
WHERE rnk = 1;   -- 공동 1위 전부 포함
```

## MAX + JOIN vs RANK() 비교

```
MAX + JOIN:
  직관적 — "최댓값 구하고 → 다시 연결"
  CTE 2개 필요
  동점 자동 처리 (AND s.sales = ms.max_sale 조건으로)

RANK():
  Window 함수 문법 알아야 함
  CTE 1개로 가능
  rnk = 1 로 동점 자동 처리 (DENSE_RANK 도 동일)

→ 둘 다 결과 동일
→ 익숙한 것 쓰면 됨
→ Window 함수 못 쓰는 환경(구버전 DB)에서는 MAX + JOIN 필수
```

## 문제 키워드 신호

```
"각 그룹에서 가장 높은/낮은 ~ 를 가진 행"
"~별 최대 판매량을 기록한 제품"
"부서별 가장 높은 급여를 받는 사원"
"사용자별 가장 최근 주문"

→ MAX + JOIN  또는  RANK() WHERE rnk = 1
→ 동점 포함 여부 확인
```