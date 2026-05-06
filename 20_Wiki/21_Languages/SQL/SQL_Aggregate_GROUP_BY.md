---
aliases:
  - GROUP BY
  - 집계함수
  - 그룹화
  - SQL 통계
  - HAVING
tags:
  - SQL
related:
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_NULL_Functions]]"
  - "[[SQL_Understanding_NULL]]"
  - "[[SQL_CASE_WHEN]]"
---
# SQL 집계 함수 · GROUP BY

## 개념 한 줄 요약

> **"흩어진 데이터를 특정 기준으로 묶어서 그룹별 통계(개수·합계·평균)를 내는 명령어."** 엑셀의 피벗 테이블을 SQL 로 구현한다고 생각하면 된다.

---

## 왜 필요한가?

"주문 내역 100만 줄" 을 그냥 보여주면 인사이트가 없다.

```
"일별 매출이 얼만데?"           → 날짜로 묶어서 SUM
"카테고리별 가장 많이 팔린 게 뭐야?" → 카테고리로 묶어서 COUNT
"팀별 평균 연봉이 얼마야?"       → 팀으로 묶어서 AVG
```

---

---

# ① GROUP BY 기본 문법

```sql
SELECT
    기준_컬럼,              -- 묶을 기준 (예: 부서명)
    COUNT(*),              -- 집계 함수 (예: 인원수)
    SUM(매출액)             -- 집계 함수 (예: 총매출)
FROM 테이블명
WHERE 조건                  -- 묶기 전 필터링 (원본 데이터 거르기)
GROUP BY 기준_컬럼           -- SELECT 에 있는 기준 컬럼은 반드시 여기도 써야 함 ⭐️
HAVING 집계_조건             -- 묶은 후 필터링 (통계치 거르기)
ORDER BY 정렬_기준;
```

---

## 다중 그룹핑 — 기준을 여러 개로 쪼개기

"부서별 인원수" 가 아닌 "부서별 + 성별 인원수" 로 세분화하려면 콤마(`,`) 로 추가하면 된다.

```
GROUP BY 부서, 성별 → 데이터가 이렇게 묶인다:

📂 영업팀
  📁 남자 → 3명
  📁 여자 → 2명
📂 인사팀
  📁 남자 → 1명
  📁 여자 → 4명
```

```sql
SELECT city, gender, COUNT(*) AS cnt
FROM customers
GROUP BY city, gender;   -- 순서대로 콤마 찍고 나열
```

|city|gender|cnt|
|---|---|---|
|서울|남|150|
|서울|여|180|
|부산|남|90|
|부산|여|110|

> ⚠️ 기준이 많아질수록 그룹이 잘게 쪼개져 **행 수는 늘고, 그룹당 데이터는 줄어든다.** 기준을 너무 많이 추가하면 "1명짜리 그룹" 만 잔뜩 생길 수 있으니 적절하게 사용한다.

## (컬럼A, 컬럼B) 쌍 단위로 묶인다 ⭐️

```
GROUP BY actor_id, director_id

→ actor_id 와 director_id 의 조합(쌍)이 같은 것끼리 묶임
→ 컬럼을 따로따로 보는 게 아니라 두 값을 함께 봄

예시 데이터:
  actor_id | director_id
     1     |     1        ← (1, 1) 쌍
     1     |     1        ← (1, 1) 쌍  → 같은 쌍끼리 묶임
     1     |     2        ← (1, 2) 쌍  → 다른 쌍
     2     |     2        ← (2, 2) 쌍  → 또 다른 쌍

GROUP BY 결과:
  (1, 1) → 2번 등장
  (1, 2) → 1번 등장
  (2, 2) → 1번 등장
```

## 실전 — 3번 이상 함께 작업한 배우-감독 쌍 찾기 ⭐️

```sql
SELECT actor_id, director_id
FROM ActorDirector
GROUP BY actor_id, director_id
HAVING COUNT(*) >= 3;
```

```
동작 순서:
  1. GROUP BY actor_id, director_id
     → (actor_id, director_id) 쌍 단위로 묶기
     → (1,1) / (1,2) / (2,2) ... 각 쌍이 하나의 그룹

  2. COUNT(*) >= 3
     → 각 쌍이 3번 이상 등장한 것만 남김
     → "이 배우와 이 감독이 3번 이상 같이 일했다"

처음에 JOIN 으로 풀려 하면:
  JOIN 은 두 테이블을 연결하는 것
  "3번 이상 함께" = 집계(COUNT) + 조건(HAVING) 으로 해결
  → GROUP BY + HAVING 패턴
```

## 다중 GROUP BY 핵심 원칙

```
GROUP BY A, B

✅ (A, B) 의 조합(쌍)이 같은 행끼리 묶임
❌ A 만 같다고 같은 그룹이 아님
❌ B 만 같다고 같은 그룹이 아님

비유:
  GROUP BY 지역, 성별
  → (서울, 남) / (서울, 여) / (부산, 남) → 전부 다른 그룹
  → 지역만 서울이 같아도 성별이 다르면 다른 그룹
```

---

---

# ② 집계 함수와 컬럼명 — 별칭을 안 쓰면 생기는 일

> **집계 함수를 쓰면 새로운 컬럼이 생긴다. 별칭을 안 붙이면 함수식 자체가 컬럼명이 된다.**

## 별칭 없을 때 vs 있을 때

```sql
-- ❌ 별칭 없음
SELECT department, SUM(salary), AVG(salary), COUNT(*)
FROM employees
GROUP BY department;
```

|department|SUM(SALARY)|AVG(SALARY)|COUNT(*)|
|---|---|---|---|
|영업팀|15000000|3000000|5|
|인사팀|9000000|3000000|3|

```sql
-- ✅ 별칭 있음
SELECT
    department,
    SUM(salary)  AS 총급여,
    AVG(salary)  AS 평균급여,
    COUNT(*)     AS 인원수
FROM employees
GROUP BY department;
```

|department|총급여|평균급여|인원수|
|---|---|---|---|
|영업팀|15000000|3000000|5|
|인사팀|9000000|3000000|3|

---

## DB 별 별칭 미지정 시 컬럼명 규칙

|DB|별칭 없을 때 컬럼명|예시|
|---|---|---|
|**Oracle**|함수식 **대문자** 그대로|`SUM(SALARY)`|
|**PostgreSQL**|함수명 **소문자**|`sum`|
|**MySQL**|함수식 그대로|`SUM(salary)`|
|**SQL Server**|컬럼명 없음 (unnamed)|`(열 이름 없음)`|

```sql
-- Oracle: 별칭 없으면 대문자 함수식
SELECT SUM(salary) FROM employees;
-- 컬럼명: SUM(SALARY)

-- Oracle: 별칭 써도 쌍따옴표 없으면 대문자로 변환
SELECT SUM(salary) AS total FROM employees;
-- 컬럼명: TOTAL

-- Oracle: 소문자 유지하려면 쌍따옴표 필수
SELECT SUM(salary) AS "total" FROM employees;
-- 컬럼명: total
```

---

## 별칭이 중요한 3가지 이유

```
① 가독성
   SUM(SALARY) 보다 총급여 가 훨씬 읽기 쉽다

② 서브쿼리·인라인 뷰에서 참조
   별칭이 없으면 컬럼명이 DB 마다 달라서 참조 시 오류가 날 수 있다

③ HAVING 에서 별칭 사용 불가 (Oracle 기준)
   HAVING 에서는 별칭이 아닌 함수식을 그대로 써야 한다
```

```sql
-- HAVING 에서 별칭 사용 불가 (Oracle / SQL Server)
SELECT department, SUM(salary) AS 총급여
FROM employees
GROUP BY department
HAVING SUM(salary) >= 10000000;   -- ✅ 함수식 그대로 써야 함
-- HAVING 총급여 >= 10000000;     -- ❌ Oracle 에서 에러

-- PostgreSQL / MySQL 은 HAVING 에서 별칭 허용
HAVING 총급여 >= 10000000;        -- ✅ PostgreSQL 에서는 가능
```

> **SQLD 시험 기준 (Oracle):** HAVING 절에서는 반드시 집계 함수식을 그대로 써야 한다.

---

---

# ③ COUNT 삼형제 + COUNT(1) ⭐️

> **가장 많이 헷갈리는 부분. 각각 무엇을 세는지 정확히 구분해야 한다.**

## 예제 테이블 (`orders`)

|row_id|user_id|비고|
|---|---|---|
|1|A||
|2|A|중복|
|3|B||
|4|C||
|5|**NULL**|🚨 비회원|

## COUNT 4가지 비교

|함수|결과|설명|NULL 처리|
|---|:-:|---|:-:|
|`COUNT(*)`|**5**|줄(Row) 수 전체|포함 ✅|
|`COUNT(1)`|**5**|줄(Row) 수 전체 (★`COUNT(*)` 와 동일)|포함 ✅|
|`COUNT(컬럼)`|**4**|해당 컬럼에 값이 있는 칸만|제외 ❌|
|`COUNT(DISTINCT 컬럼)`|**3**|중복 제거 + NULL 제거 (A, B, C)|제외 ❌|

---

## COUNT(1) vs COUNT(*) — 왜 같은가?

```
COUNT(*) → "이 행이 존재하는가?" 를 체크. 행 자체를 카운트.
COUNT(1) → "이 행마다 숫자 1을 붙여라" 를 카운트.

두 경우 모두 NULL 여부와 관계없이 행 자체의 존재만 보기 때문에
결과가 완벽히 동일하다.
```

```sql
SELECT COUNT(*)  FROM orders;   -- 결과: 5
SELECT COUNT(1)  FROM orders;   -- 결과: 5  (완전히 같음)
SELECT COUNT(2)  FROM orders;   -- 결과: 5  (숫자는 뭐든 상관없음)
SELECT COUNT('X') FROM orders;  -- 결과: 5  (문자열도 동일)
```

> **왜 COUNT(1) 을 쓰는가?** 예전에 일부 DB 엔진에서 `COUNT(*)` 보다 `COUNT(1)` 이 빠르다는 속설이 있었다. **현재 대부분의 DB (PostgreSQL · Oracle · MySQL) 에서는 옵티마이저가 동일하게 처리하므로 성능 차이 없다.** 코드 스타일이나 팀 컨벤션에 따라 선택하면 된다.

## 상황별 COUNT 선택 가이드

|목적|추천 함수|
|---|---|
|전체 주문 건수 · 행 수|`COUNT(*)` 또는 `COUNT(1)`|
|활동 중인 회원 수 (NULL 제외)|`COUNT(user_id)`|
|순수 회원 수 · 고유 값 수|`COUNT(DISTINCT user_id)`|

---

---

# ③ 기타 집계 함수 — SUM · AVG · MIN · MAX

> **NULL 을 "없는 셈" 으로 처리한다. 0이 아니라 아예 투명인간 취급.**

## 예제 데이터: `[100, 200, NULL]` (총 3건)

|함수|계산 과정|결과|주의사항|
|---|---|:-:|---|
|`SUM(컬럼)`|100 + 200|**300**|NULL을 0으로 더한 것과 결과는 같음|
|`AVG(컬럼)`|(100 + 200) / **2**|**150**|**÷3 이 아니라 ÷2!** NULL 행을 분모에서 제외|
|`MIN(컬럼)`|100, 200 중 최소|**100**|NULL은 비교 대상 제외|
|`MAX(컬럼)`|100, 200 중 최대|**200**|NULL은 비교 대상 제외|

> **AVG 함정:** NULL 을 0으로 포함해서 평균을 내고 싶다면?

```sql
AVG(COALESCE(amount, 0))   -- NULL → 0 으로 바꾼 후 평균
```

> → [[SQL_NULL_Functions]] 참고

## ⭐️ 예외 — 결과 자체가 NULL 이 되는 경우

> **더할 대상이 아예 없거나, 대상이 모두 NULL 뿐일 때는 결과값이 NULL 로 반환된다.** 0 이 아니라 NULL 이라는 점이 핵심.

```sql
-- 케이스 ①: 조건에 맞는 행 자체가 없을 때
SELECT SUM(salary) FROM employees WHERE department = '없는팀';
-- 결과: NULL  (0 이 아님!)

-- 케이스 ②: 대상이 전부 NULL 일 때
-- 데이터: [NULL, NULL, NULL]
SELECT SUM(salary)  FROM employees;  -- 결과: NULL
SELECT AVG(salary)  FROM employees;  -- 결과: NULL
SELECT MIN(salary)  FROM employees;  -- 결과: NULL
SELECT MAX(salary)  FROM employees;  -- 결과: NULL
```

|상황|COUNT(*)|SUM / AVG / MIN / MAX|
|---|:-:|:-:|
|행이 아예 없을 때|**0**|**NULL**|
|대상 컬럼이 전부 NULL 일 때|**행 수 그대로**|**NULL**|

```
COUNT(*) 는 행의 존재 자체를 세므로 → 0 반환
SUM · AVG · MIN · MAX 는 더할 값이 없으므로 → NULL 반환
```

> **실무 주의:** 집계 결과를 바로 연산에 쓸 때 NULL 이 나오면 전파된다.

```sql
 -- SUM 이 NULL 이면 이 계산 전체가 NULL
SELECT SUM(bonus) + 1000 FROM employees;  -- SUM = NULL 이면 결과도 NULL

-- 안전하게: COALESCE 로 NULL → 0 처리
SELECT COALESCE(SUM(bonus), 0) + 1000 FROM employees;
```

---

---

# ④ WHERE vs HAVING — 가장 중요한 차이

|구분|WHERE|HAVING|
|---|---|---|
|**시점**|그룹핑 **전**|그룹핑 **후**|
|**대상**|원본 데이터 (Raw Data)|집계된 데이터|
|**집계함수**|❌ 사용 불가|✅ 사용 가능|
|**예시**|"서울 사는 사람만 뽑아서"|"평균 매출 100만 이상인 팀만"|

```sql
SELECT user_id, AVG(amount) AS avg_spent
FROM orders
WHERE city = '서울'          -- 그룹핑 전: 서울 데이터만 필터링
GROUP BY user_id
HAVING AVG(amount) >= 100000; -- 그룹핑 후: 평균 10만 이상인 고객만
```

> **한 줄 요약:** "원본을 거를 땐 WHERE, 계산된 결과를 거를 땐 HAVING"

## HAVING 안에 쓸 수 있는 것들

```sql
HAVING COUNT(*)          >= 10          -- 집계함수 + 비교 연산자
HAVING SUM(amount)       >= 100000
HAVING AVG(score)        >= 80
HAVING MAX(price)        < 50000
HAVING MIN(rating)       >= 4.0

HAVING COUNT(DISTINCT category) >= 2    -- DISTINCT 조합

HAVING COUNT(DISTINCT                   -- CASE WHEN 조합 ⭐
    CASE WHEN 조건 THEN 값 END
) >= 2

HAVING SUM(amount) >= AVG(amount) * 2   -- 집계함수끼리 계산도 가능
```

---

---

# ⑤ RANK vs COUNT 구분 — 실수 방지

> **"순위 번호가 결과에 직접 필요한가?"** 로 판단한다.

```
순위 번호(1등, 2등, 3등...)가 필요한가?
        ↓ YES                    ↓ NO
     RANK 써라          COUNT(*) + GROUP BY + HAVING
```

|문제 유형|핵심 질문|정답|
|---|---|---|
|판매량 1위 작품은?|순위 번호 필요|✅ RANK|
|1위~3위 목록|순위 번호 필요|✅ RANK|
|상위 50위 이내 2회 이상 등재|**등재 횟수** 필요|✅ COUNT(*)|
|TOP 10에 3번 이상 든 작가|**등재 횟수** 필요|✅ COUNT(*)|

> **문제에 "몇 번", "횟수", "등재" 라는 단어가 보이면 → RANK 전에 COUNT 먼저 생각.**

---

---

# ⑥ 자주 하는 실수 4가지

## 실수 1 — not a GROUP BY expression 에러

```sql
-- ❌ 에러: name이 SELECT 에 있는데 GROUP BY 에 없음
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;

-- ✅ 해결: 집계함수에 감싸거나 GROUP BY 에 추가
SELECT department, COUNT(*)
FROM employees
GROUP BY department;
```

> **법칙:** SELECT 절의 컬럼은 집계함수로 감싸져 있거나, GROUP BY 에 반드시 포함되어야 한다.

---

## 실수 2 — 중첩 집계 (AVG(SUM)) 시도

```sql
-- ❌ 에러: Group function is nested too deeply
SELECT AVG(SUM(total_bill))
FROM tips
GROUP BY day;

-- ✅ 해결: 서브쿼리로 단계 분리
SELECT AVG(daily_total)
FROM (
    SELECT day, SUM(total_bill) AS daily_total   -- 1단계: 일별 합계
    FROM tips
    GROUP BY day
) t;                                              -- 2단계: 합계의 평균
```

> SQL 은 한 번의 단계에서 **오직 한 번의 집계만** 가능하다.

---

## 실수 3 — ID를 SUM 으로 더하기

```
"고객이 몇 번 대여했는지" 를 구하라 → SUM(customer_id) ❌

customer_id 는 사람을 구별하는 이름표(식별자). 계산 가능한 수량이 아니다.
학번 1번 + 학번 2번 = 3명? → 말이 안 됨.

"~한 횟수", "~한 건수", "몇 번?" → COUNT (행의 개수)
"총 매출", "총합", "전체 금액"   → SUM (값의 크기)
```

---

## 실수 4 — GROUP BY 의 NULL 처리

```sql
SELECT department, COUNT(*) AS cnt
FROM employees
GROUP BY department;
```

|department|cnt|
|---|---|
|영업팀|5|
|인사팀|3|
|**NULL**|**2** ← 부서 미배정 직원도 하나의 그룹으로 출력됨!|

> GROUP BY 는 NULL 끼리 같은 그룹으로 묶어서 결과에 포함시킨다. NULL 그룹을 제외하려면 **WHERE 로 먼저 거르는 게 성능상 유리하다.**

```sql
WHERE department IS NOT NULL   -- GROUP BY 전에 NULL 제거 (권장)
-- HAVING department IS NOT NULL  -- 가능하지만 성능상 비효율
```

---

## 실수 5 (시험 함정) — HAVING 단독 사용 가능 여부

```
"HAVING 은 반드시 GROUP BY 와 함께 써야 한다" → ❌ 틀린 말!

GROUP BY 없이 HAVING 만 쓰면 → 테이블 전체를 하나의 그룹으로 취급해서 실행된다.
실무에서 거의 안 쓰지만 문법적으로는 가능하다.

SQLD 에서 "HAVING 은 반드시 GROUP BY 와 함께 써야 한다(X)" 형태로 낚시 보기로 자주 출제.
```

---

---
