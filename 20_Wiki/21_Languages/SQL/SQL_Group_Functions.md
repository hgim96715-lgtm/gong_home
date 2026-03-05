---
aliases:
  - 그룹 함수
  - ROLLUP
  - CUBE
  - GROPING SETS
  - 소계
  - 총계
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
---
# SQL 그룹 함수 (Group Functions)

## 개념 한 줄 요약

> **"소계(Subtotal)와 총계(Grand Total)를 한 번의 쿼리로 자동 계산하는 다차원 집계 함수."**

---

## 왜 필요한가?

`GROUP BY` 만으로는 하나의 기준으로만 집계할 수 있다. 지역별 합계 + 상품별 합계 + 전체 총계를 모두 보려면 원래 `UNION ALL` 을 여러 번 써야 했다.

```sql
-- ❌ UNION ALL 방식: 쿼리가 길고 테이블을 여러 번 스캔해서 성능 나쁨
SELECT 지역, 상품, SUM(가격) FROM 판매 GROUP BY 지역, 상품
UNION ALL
SELECT 지역, NULL,  SUM(가격) FROM 판매 GROUP BY 지역
UNION ALL
SELECT NULL,  NULL,  SUM(가격) FROM 판매;
```

그룹 함수를 쓰면 **테이블을 한 번만 스캔** 해서 모든 레벨의 집계를 한 번에 뽑아낸다.

---

## 3대 그룹 함수 한눈에 비교

|함수|생성 집계 수|순서 영향|특징|
|---|---|---|---|
|`ROLLUP(A, B)`|n+1 개|✅ **있음**|오른쪽→왼쪽 계층적 소계|
|`CUBE(A, B)`|2ⁿ 개|❌ 없음|가능한 모든 조합의 소계|
|`GROUPING SETS(A, B)`|지정한 개수만큼|❌ 없음|원하는 조합만 콕 집어서|

> **공통 예제 데이터** (이하 모든 예제에서 공통 사용)

|지역|상품|가격|
|---|---|---|
|서울|A|250|
|서울|B|300|
|부산|A|550|
|부산|B|350|

---

---

# ① ROLLUP — 계층적 소계 (n+1 개)

오른쪽에서 왼쪽으로 컬럼을 하나씩 제거하며 소계와 총계를 계산한다. **인자 순서가 바뀌면 결과도 달라진다.**

## ROLLUP(지역, 상품) — 인자 2개 → 3가지 집계

```sql
SELECT 지역, 상품, SUM(가격) AS 총합
FROM 판매
GROUP BY ROLLUP(지역, 상품);
```

```
생성되는 그룹:
① (지역, 상품) → 상세 데이터
② (지역)       → 지역별 소계  (상품 컬럼 제거)
③ ()           → 전체 총계    (지역, 상품 모두 제거)
```

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|A|250|① 상세|
|서울|B|300|① 상세|
|**서울**|**NULL**|**550**|**② 서울 소계**|
|부산|A|550|① 상세|
|부산|B|350|① 상세|
|**부산**|**NULL**|**900**|**② 부산 소계**|
|**NULL**|**NULL**|**1450**|**③ 전체 총계**|

---

## ROLLUP(job) — 인자 1개 → 2가지 집계 ⭐ SQLD 단골

```sql
SELECT job, SUM(sal) AS 총합
FROM emp
GROUP BY ROLLUP(job);
```

```
생성되는 그룹:
① (job)  → job 별 합계
② ()     → 전체 총계 (소계 없이 총계 1줄만 추가)
```

> **핵심:** 인자가 1개면 n+1 = 2 가지. **소계 없이 전체 총계 한 줄만 추가** 된다.

|job|총합|그룹화 기준|
|---|---|---|
|CLERK|4150|① 상세|
|MANAGER|8275|① 상세|
|ANALYST|6000|① 상세|
|PRESIDENT|5000|① 상세|
|SALESMAN|5600|① 상세|
|**NULL**|**29025**|**② 전체 총계**|

---

## 부분 ROLLUP — 전체 총계를 없애고 싶을 때

```sql
-- A 는 고정, B 에 대해서만 소계를 낸다
GROUP BY 지역, ROLLUP(상품);
```

```
생성되는 그룹:
① (지역, 상품) → 상세
② (지역)       → 지역별 소계
              ← 전체 총계 없음! (지역이 괄호 밖에 있으므로)
```

---

---

# ② CUBE — 모든 조합 (2ⁿ 개)

가능한 모든 그룹 조합에 대해 집계한다. **인자 순서를 바꿔도 결과 집합은 동일하다.** `ROLLUP` 에는 없는 **"상품별 소계"** 가 추가되는 게 핵심 차이다.

```sql
SELECT 지역, 상품, SUM(가격) AS 총합
FROM 판매
GROUP BY CUBE(지역, 상품);
```

```
생성되는 그룹:
① (지역, 상품) → 상세
② (지역)       → 지역별 소계
③ (상품)       → 상품별 소계 ← ROLLUP 에는 없음!
④ ()           → 전체 총계
```

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|A|250|① 상세|
|서울|B|300|① 상세|
|**서울**|**NULL**|**550**|**② 지역 소계**|
|부산|A|550|① 상세|
|부산|B|350|① 상세|
|**부산**|**NULL**|**900**|**② 지역 소계**|
|**NULL**|**A**|**800**|**③ 상품 A 소계 ← ROLLUP 에 없음!**|
|**NULL**|**B**|**650**|**③ 상품 B 소계 ← ROLLUP 에 없음!**|
|**NULL**|**NULL**|**1450**|**④ 전체 총계**|

---

---

# ③ GROUPING SETS — 원하는 것만 지정

총계나 모든 소계를 강제로 계산하지 않고, **내가 명시한 그룹만** 평면적으로 집계한다.

```sql
SELECT 지역, 상품, SUM(가격) AS 총합
FROM 판매
GROUP BY GROUPING SETS(지역, 상품);
-- 지역별 합계 + 상품별 합계, 딱 2가지만. 전체 총계 없음.
```

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|NULL|550|① 지역별|
|부산|NULL|900|① 지역별|
|NULL|A|800|② 상품별|
|NULL|B|650|② 상품별|

## GROUPING SETS 안에 ROLLUP 혼합

```sql
GROUP BY GROUPING SETS(지역, ROLLUP(상품));
```

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|NULL|550|① 지역|
|부산|NULL|900|① 지역|
|NULL|A|800|② ROLLUP 상품별|
|NULL|B|650|② ROLLUP 상품별|
|**NULL**|**NULL**|**1450**|**③ ROLLUP 이 만드는 전체 총계**|

---

---

# ④ GROUPING 함수 — NULL 을 텍스트 이름표로 바꾸기

## 왜 GROUPING 함수가 필요한가?

ROLLUP · CUBE 가 소계·총계 행을 만들 때, 그룹 기준이 되지 않은 컬럼 자리에 `NULL` 을 채운다.

```
지역    상품    총합
서울    NULL    550   ← 이 NULL 이 뭔지 모름
NULL    NULL    1450  ← 이 NULL 도 뭔지 모름
```

그런데 원본 데이터 자체에도 `NULL` 이 있을 수 있다.

```
지역    상품    총합
서울    NULL    550   ← ROLLUP 이 만든 NULL? 아니면 원래 NULL 상품?
```

> **이 두 가지 NULL 을 구별하는 유일한 방법이 `GROUPING` 함수다.**

---

## GROUPING 작동 원리

```
GROUPING(컬럼) 은 해당 행이 그 컬럼을 기준으로 "집계된 행"인지 여부를 반환한다.

반환값 1 → ROLLUP/CUBE 가 소계·총계를 위해 그 컬럼을 뭉개서 만든 행
            = 집계 과정에서 생긴 가짜 NULL

반환값 0 → 원본 데이터에 실제로 존재하는 값을 가진 행
            = 진짜 데이터 행
```

### 직관적으로 이해하기

```
ROLLUP(지역, 상품) 결과:

지역    상품    GROUPING(지역)  GROUPING(상품)  의미
서울    A       0               0               실제 데이터 행
서울    B       0               0               실제 데이터 행
서울    NULL    0               1               지역은 실제값, 상품은 집계로 뭉갠 것 → 서울 소계
부산    A       0               0               실제 데이터 행
부산    B       0               0               실제 데이터 행
부산    NULL    0               1               부산 소계
NULL    NULL    1               1               둘 다 집계로 뭉갠 것 → 전체 총계
```

> **핵심:** `GROUPING(컬럼) = 1` 이면 그 컬럼이 집계를 위해 "사라진(뭉개진)" 행이다. 쉽게 말해 **"이 행은 그 컬럼을 기준으로 묶인 합산 행이야"** 라는 표시다.

---

## 실전 사용법 — CASE WHEN 과 조합

```sql
SELECT
    CASE WHEN GROUPING(지역) = 1 THEN '전체합계' ELSE 지역 END AS 지역,
    CASE WHEN GROUPING(상품) = 1 THEN '소계'     ELSE 상품 END AS 상품,
    SUM(가격) AS 총합
FROM 판매
GROUP BY ROLLUP(지역, 상품);
```

|지역|상품|총합|GROUPING(지역)|GROUPING(상품)|
|---|---|---|:-:|:-:|
|서울|A|250|0|0|
|서울|B|300|0|0|
|서울|**소계**|550|0|**1**|
|부산|A|550|0|0|
|부산|B|350|0|0|
|부산|**소계**|900|0|**1**|
|**전체합계**|**소계**|1450|**1**|**1**|

> **마지막 행(전체 총계)** 에서 지역 = `'전체합계'`, 상품 = `'소계'` 로 나오는 이유: 두 컬럼 모두 `GROUPING = 1` 이기 때문에 각각의 CASE 가 작동한 결과다. 컬럼명을 다르게 하고 싶다면 아래처럼 두 컬럼을 동시에 보고 판단해야 한다.

```sql
-- 총계 행을 더 명확하게 표현하려면
CASE
    WHEN GROUPING(지역) = 1 AND GROUPING(상품) = 1 THEN '전체 총계'
    WHEN GROUPING(지역) = 0 AND GROUPING(상품) = 1 THEN 지역 || ' 소계'
    ELSE 지역
END AS 지역명
```

---

---

# ⑤ 부분 ROLLUP · CUBE — 전체 총계 유무 구분 ⭐ SQLD 킬러

> **`ROLLUP(A, B)` 처럼 괄호 안에 모두 넣어야 전체 총계가 생긴다.** `A, ROLLUP(B)` 처럼 A 를 밖에 빼면 A 는 항상 고정 그룹이 되어 전체 총계가 없다.

|작성 방식|생성되는 집계|전체 총계?|
|---|---|---|
|`ROLLUP(grade, job)`|(grade,job) + (grade 소계) + **(전체 총계)**|✅ **있음**|
|`grade, ROLLUP(job)`|(grade,job) + (grade 소계)|❌ **없음**|
|`grade, CUBE(job)`|(grade,job) + (grade 소계)|❌ **없음**|

---

## SQLD 기출 문제 풀이

```sql
SELECT b.grade, a.job, SUM(a.sal) AS SUM_SAL, COUNT(*) AS CNT
FROM emp a, salgrade b
WHERE a.sal BETWEEN b.losal AND b.hisal
GROUP BY ㉠;
```

**실행 결과:**

|GRADE|JOB|SUM_SAL|CNT|
|---|---|---|---|
|2|CLERK|1300|1|
|2|SALESMAN|2500|2|
|**2**|**NULL**|**3800**|**3**|
|3|SALESMAN|1500|1|
|**3**|**NULL**|**1500**|**1**|
|4|ANALYST|6000|2|
|4|MANAGER|5425|2|
|**4**|**NULL**|**11425**|**4**|

**결과 패턴 분석:**

```
(grade, job) 상세 행     → ✅ 있음
(grade) 소계 행          → ✅ 있음 (JOB = NULL)
(전체 총계) 행           → ❌ 없음 (GRADE = NULL 인 행이 없다)
```

**보기 분석:**

|보기|생성 집계|전체 총계|정답?|
|---|---|---|---|
|① `GROUPING SETS(grade, (job, grade))`|(grade) + (grade,job)|없음|✅ 일치|
|**② `ROLLUP(grade, job)`**|**(grade,job) + (grade소계) + (전체총계)**|**있음**|**❌ 불일치**|
|③ `grade, ROLLUP(job)`|(grade,job) + (grade소계)|없음|✅ 일치|
|④ `grade, CUBE(job)`|(grade,job) + (grade소계)|없음|✅ 일치|

> **정답: ② ROLLUP(grade, job)** `ROLLUP(grade, job)` 은 전체 총계 행(GRADE=NULL, JOB=NULL) 이 추가로 생성되어 주어진 결과와 다르다.


---

## ORDER BY 사용

> **ROLLUP · CUBE · GROUPING SETS 결과에 정렬이 필요한 경우 ORDER BY 절을 사용할 수 있다.**

```sql
-- ROLLUP 후 정렬
SELECT 지역, 상품, SUM(매출) AS 총매출
FROM 판매
GROUP BY ROLLUP(지역, 상품)
ORDER BY 지역, 상품;

-- CUBE 후 정렬
SELECT 지역, 상품, SUM(매출) AS 총매출
FROM 판매
GROUP BY CUBE(지역, 상품)
ORDER BY 지역 NULLS LAST, 상품 NULLS LAST;
-- NULLS LAST: 소계·총계 행(NULL)을 맨 아래로 밀기

-- GROUPING SETS 후 정렬
SELECT 지역, 상품, SUM(매출) AS 총매출
FROM 판매
GROUP BY GROUPING SETS(지역, 상품)
ORDER BY 총매출 DESC;
```

> **`NULLS LAST` 활용 팁:** ROLLUP · CUBE 가 만드는 소계·총계 행은 그룹 컬럼이 NULL 이라서 기본 정렬 시 NULL 이 맨 위로 올라올 수 있다. `ORDER BY 컬럼 NULLS LAST` 를 쓰면 소계·총계 행을 맨 아래에 배치할 수 있다.

---

## 관련 노트

- [[SQL_GROUP_BY_HAVING]] — GROUP BY · HAVING 기본
- [[SQL_Aggregate_Functions]] — COUNT · SUM · AVG · MAX · MIN
- [[SQL_CASE_WHEN]] — GROUPING 함수와 CASE WHEN 조합
- [[SQL_Window_Functions]] — ROW_NUMBER · RANK · DENSE_RANK