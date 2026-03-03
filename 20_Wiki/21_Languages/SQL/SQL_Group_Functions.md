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

> 소계(Subtotal)와 총계(Grand Total)를 한 번의 쿼리로 자동 계산하는 다차원 집계 함수

---

## 왜 필요한가?

`GROUP BY`만으로는 **하나의 기준**으로만 집계할 수 있다.  
지역별 합계 + 상품별 합계 + 전체 총계를 모두 보려면 원래 `UNION ALL`을 여러 번 써야 했다.

```sql
-- ❌ UNION ALL 방식: 쿼리가 길고 성능도 나쁨
SELECT 지역, 상품, SUM(가격) FROM 판매 GROUP BY 지역, 상품
UNION ALL
SELECT 지역, NULL,  SUM(가격) FROM 판매 GROUP BY 지역
UNION ALL
SELECT NULL,  NULL,  SUM(가격) FROM 판매;
```

그룹 함수를 사용하면 **테이블을 한 번만 스캔**해서 모든 레벨의 집계를 한 번에 뽑아낸다.

---

## 3대 그룹 함수 한눈에 비교

|함수|생성되는 집계 수|순서 영향|특징|
|---|---|---|---|
|`ROLLUP(A, B)`|n+1개|**있음**|오른쪽→왼쪽으로 계층적 소계|
|`CUBE(A, B)`|2ⁿ개|없음|가능한 모든 조합의 소계|
|`GROUPING SETS(A, B)`|지정한 개수만큼|없음|원하는 조합만 콕 집어서|

> **기본 예제 데이터** (이하 모든 예제에서 공통 사용)

|지역|상품|가격|
|---|---|---|
|서울|A|250|
|서울|B|300|
|부산|A|550|
|부산|B|350|

---

## ROLLUP — 계층적 소계 (n+1개)

오른쪽에서 왼쪽으로 컬럼을 하나씩 제거하며 소계와 총계를 계산한다.  
**인자 순서가 바뀌면 결과도 달라진다.**

### 인자가 2개인 경우: ROLLUP(지역, 상품) → 3가지 집계

```sql
SELECT 지역, 상품, SUM(가격) AS 총합
FROM 판매
GROUP BY ROLLUP(지역, 상품);
```

생성되는 그룹: `(지역, 상품)` → `(지역)` → `()` 총 **3가지**

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|A|250|① (지역, 상품)|
|서울|B|300|① (지역, 상품)|
|**서울**|**NULL**|**550**|**② (지역) 서울 소계**|
|부산|A|550|① (지역, 상품)|
|부산|B|350|① (지역, 상품)|
|**부산**|**NULL**|**900**|**② (지역) 부산 소계**|
|**NULL**|**NULL**|**1450**|**③ () 전체 총계**|

---

### 인자가 1개인 경우: ROLLUP(job) → 2가지 집계 ★ 시험 단골

```sql
SELECT job, SUM(sal) AS 총합
FROM emp
GROUP BY ROLLUP(job);
```

생성되는 그룹: `(job)` → `()` 총 **2가지**

> **핵심:** 인자가 1개면 `n+1 = 1+1 = 2`가지. job별 합계 + 전체 총계만 나온다.  
> 소계 없이 **총계 한 줄만 추가**되는 것이 포인트.

|job|총합|그룹화 기준|
|---|---|---|
|CLERK|4150|① (job)|
|MANAGER|8275|① (job)|
|ANALYST|6000|① (job)|
|PRESIDENT|5000|① (job)|
|SALESMAN|5600|① (job)|
|**NULL**|**29025**|**② () 전체 총계**|

```sql
-- 부분 ROLLUP: 지역은 고정, 상품에 대해서만 소계+총계
GROUP BY 지역, ROLLUP(상품);
-- 생성: (지역, 상품) + (지역 소계) → 지역별 소계는 있지만 전체 총계는 없음
```

---

## 4. CUBE — 모든 조합 (2ⁿ개)

가능한 모든 그룹 조합에 대해 집계한다. `ROLLUP`에는 없는 **"상품별 소계"** 가 추가된다.  
**인자 순서를 바꿔도 결과는 동일하다.**

```sql
SELECT 지역, 상품, SUM(가격) AS 총합
FROM 판매
GROUP BY CUBE(지역, 상품);
```

생성되는 그룹: `(지역, 상품)` → `(지역)` → `(상품)` → `()` 총 **4가지**

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|A|250|① (지역, 상품)|
|서울|B|300|① (지역, 상품)|
|**서울**|**NULL**|**550**|**② (지역) 서울 소계**|
|부산|A|550|① (지역, 상품)|
|부산|B|350|① (지역, 상품)|
|**부산**|**NULL**|**900**|**② (지역) 부산 소계**|
|**NULL**|**A**|**800**|**③ (상품) A 소계 ← ROLLUP엔 없음!**|
|**NULL**|**B**|**650**|**③ (상품) B 소계 ← ROLLUP엔 없음!**|
|**NULL**|**NULL**|**1450**|**④ () 전체 총계**|

---

## GROUPING SETS — 원하는 것만 (지정한 개수만큼)

총계나 모든 소계를 강제로 계산하지 않고, **내가 명시한 그룹만** 평면적으로 집계한다.

```sql
SELECT 지역, 상품, SUM(가격) AS 총합
FROM 판매
GROUP BY GROUPING SETS(지역, 상품);
```

지역별 합계 + 상품별 합계 딱 **2가지만** 출력. 전체 총계 없음.

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|NULL|550|① (지역) 서울 합계|
|부산|NULL|900|① (지역) 부산 합계|
|NULL|A|800|② (상품) A 합계|
|NULL|B|650|② (상품) B 합계|

### GROUPING SETS 안에 ROLLUP/CUBE 혼합

```sql
-- 지역별 합계 + 상품별 합계 + 전체 총계를 한 번에
GROUP BY GROUPING SETS(지역, ROLLUP(상품));
```

|지역|상품|총합|그룹화 기준|
|---|---|---|---|
|서울|NULL|550|① 지역 그룹화|
|부산|NULL|900|① 지역 그룹화|
|NULL|A|800|② ROLLUP(상품)의 상품별 합계|
|NULL|B|650|② ROLLUP(상품)의 상품별 합계|
|**NULL**|**NULL**|**1450**|**③ ROLLUP이 만드는 전체 총계**|

---

##  GROUPING 함수 — NULL을 예쁜 이름표로 바꾸기

소계/총계 자리에 찍히는 `NULL`을 "소계", "총합계" 같은 텍스트로 바꿀 때 사용한다.

|GROUPING(컬럼) 반환값|의미|
|---|---|
|`1`|집계 함수가 만든 **가짜 NULL** (소계/총계 행)|
|`0`|원본 데이터의 **진짜 값**|

```sql
SELECT 
    CASE WHEN GROUPING(지역) = 1 THEN '총합계' ELSE 지역 END AS 지역,
    CASE WHEN GROUPING(상품) = 1 THEN '소계'   ELSE 상품 END AS 상품,
    SUM(가격) AS 총합
FROM 판매
GROUP BY ROLLUP(지역, 상품);
```

|지역|상품|총합|
|---|---|---|
|서울|A|250|
|서울|B|300|
|서울|**소계**|550|
|부산|A|550|
|부산|B|350|
|부산|**소계**|900|
|**총합계**|**소계**|1450|

---

## 부분 ROLLUP / CUBE (시험 핵심 ★★★)

`GROUP BY` 뒤에 **일반 컬럼과 ROLLUP/CUBE를 섞어 쓰면** 전체 총계가 생기지 않는다.  
이 차이가 시험에서 "결과가 다른 것 고르기" 유형으로 자주 출제된다.

|작성 방식|생성되는 집계|전체 총계?|
|---|---|---|
|`ROLLUP(grade, job)`|(grade,job) + (grade 소계) + **(전체 총계)**|**있음**|
|`grade, ROLLUP(job)`|(grade,job) + (grade 소계)|**없음**|
|`grade, CUBE(job)`|(grade,job) + (grade 소계)|**없음** (CUBE 인자 1개 = ROLLUP과 동일)|

> **핵심 규칙:** `ROLLUP(A, B)` 처럼 **괄호 안에 모두** 넣어야 전체 총계`()`가 생긴다.  
> `A, ROLLUP(B)` 처럼 **A를 밖에 빼면** A는 항상 고정 그룹이 되어 전체 총계가 만들어지지 않는다.

### SQLD 기출 문제

```sql
SELECT b.grade, a.job, SUM(a.sal) AS SUM_SAL, COUNT(*) AS CNT
FROM emp a, salgrade b
WHERE a.sal BETWEEN b.losal AND b.hisal
GROUP BY ㉠
```

**실행 결과**

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

결과의 패턴을 분석하면:

- `(grade, job)` 상세 행 → ✅ 있음
- `(grade)` 소계 행 (JOB=NULL) → ✅ 있음
- `(전체 총계)` 행 (GRADE=NULL, JOB=NULL) → ❌ **없음**

**보기 분석**

|보기|생성 집계|전체 총계|결과 일치?|
|---|---|---|---|
|① `GROUPING SETS(grade, (job, grade))`|(grade) + (grade, job)|없음|✅ 일치|
|**② `ROLLUP(grade, job)`**|**(grade,job) + (grade 소계) + (전체 총계)**|**있음**|**❌ 불일치**|
|③ `grade, ROLLUP(job)`|(grade, job) + (grade 소계)|없음|✅ 일치|
|④ `grade, CUBE(job)`|(grade, job) + (grade 소계)|없음|✅ 일치|

> **정답: ② ROLLUP(grade, job)**  
> `ROLLUP(grade, job)`은 전체 총계 행(GRADE=NULL, JOB=NULL)이 추가로 생성되어 주어진 결과와 다르다.

**② 번이 만들어낼 실제 결과 (차이점: 마지막 행)**

|GRADE|JOB|SUM_SAL|CNT|
|---|---|---|---|
|2|CLERK|1300|1|
|2|SALESMAN|2500|2|
|2|NULL|3800|3|
|3|SALESMAN|1500|1|
|3|NULL|1500|1|
|4|ANALYST|6000|2|
|4|MANAGER|5425|2|
|4|NULL|11425|4|
|**NULL**|**NULL**|**17425**|**8** ← 이 행이 추가됨|

---

## 핵심 정리 & 실수 주의

### 인자 개수별 생성 집계 수

|함수|인자 1개|인자 2개|인자 3개|
|---|---|---|---|
|ROLLUP|2가지 (n+1)|3가지|4가지|
|CUBE|2가지 (2ⁿ)|4가지|8가지|

### 자주 나오는 실수

**"ROLLUP(A, B)와 ROLLUP(B, A)는 같다?"**  
→ **다르다.** ROLLUP은 순서가 바뀌면 소계 기준이 바뀐다.  
→ CUBE와 GROUPING SETS는 순서가 바뀌어도 결과 집합이 동일하다.

**"NULL이 보이면 무조건 소계/총계 행이다?"**  
→ **아니다.** 원본 데이터 자체에 NULL이 있을 수 있다.  
→ 진짜 NULL과 집계용 가짜 NULL을 구별하는 유일한 방법이 `GROUPING` 함수다.

**"ROLLUP(job) 결과에 소계가 없다?"**  
→ **맞다.** 인자가 1개면 소계 단계가 없고 **전체 총계 1줄만 추가**된다.

**"grade, ROLLUP(job)과 ROLLUP(grade, job)은 같다?"**  
→ **다르다.** `ROLLUP(grade, job)`은 전체 총계(GRADE=NULL) 행이 생기지만, `grade, ROLLUP(job)`은 grade가 항상 고정이라 전체 총계가 생기지 않는다.