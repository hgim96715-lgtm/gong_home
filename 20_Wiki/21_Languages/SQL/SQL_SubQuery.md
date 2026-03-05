---
aliases:
  - 서브쿼리
  - SubQuery
  - 스칼라 서브쿼리
  - 인라인 뷰
  - 중첩 서브쿼리
tags:
  - SQL
related:
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_DISTINCT_vs_GROUP_BY]]"
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_Standard_JOIN]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_WITH_CTE]]"
  - "[[SQL_View]]"
---
# SQL 서브쿼리 (SubQuery)

## 개념 한 줄 요약

> **"쿼리 안의 쿼리. 괄호 `()` 안에서 먼저 실행되어 메인 쿼리를 보조하는 하위 쿼리."**

---

## 왜 필요한가?

"평균 급여보다 많이 받는 사람을 찾아줘" 라고 할 때, 평균을 구하고 → 메모하고 → 다시 쿼리를 짜는 번거로움을 없애준다. 서브쿼리를 쓰면 DB 가 내부 쿼리를 먼저 실행하고, 그 결과로 메인 쿼리를 한 번에 처리한다.

---

---

# ① 위치에 따른 3가지 분류

> 서브쿼리는 **"어느 절(Clause)에 위치하느냐"** 에 따라 이름과 규칙이 완전히 달라진다.

|위치|이름|특징|
|---|---|---|
|`SELECT` 절|스칼라 서브쿼리|컬럼처럼 동작. **반드시 1행 1열** 반환|
|`FROM` 절|인라인 뷰|가상 테이블처럼 동작. **별칭 필수**|
|`WHERE` / `HAVING` 절|중첩 서브쿼리|조건 필터링. 결과 형태에 따라 연산자 달라짐|

---

## A. 스칼라 서브쿼리 — SELECT 절

> **반드시 1행 1열(단일 값)** 만 반환해야 한다. 컬럼 자리에 들어가는 모든 곳에 사용 가능.

```sql
-- ① SELECT 절: 옆에 회사 평균 급여를 나란히 보여주기 (VLOOKUP 스타일)
SELECT
    직원명,
    급여,
    (SELECT AVG(급여) FROM 직원) AS 전체평균급여   -- 컬럼처럼 동작
FROM 직원;

-- ② UPDATE SET 절: 계산된 값으로 일괄 업데이트
UPDATE 부서
SET 최고급여 = (
    SELECT MAX(급여)
    FROM 직원
    WHERE 직원.부서코드 = 부서.부서코드
)
WHERE 부서코드 = 100;

-- ③ WHERE 절: 동적인 기준값으로 필터링
SELECT 직원명 FROM 직원
WHERE 급여 > (SELECT AVG(급여) FROM 직원);
```

> **스칼라 서브쿼리는 SELECT 절에서만 쓰는 게 아니다.** 컬럼명이나 상수값이 올 수 있는 자리라면 어디든 들어갈 수 있다.

---

## B. 인라인 뷰 — FROM 절

> **가상 테이블처럼 동작.** 메모리에 임시로 올려두고 쓰기 때문에 **별칭이 반드시 필요하다.**

```sql
SELECT
    e.직원명,
    t.부서_최대급여
FROM 직원 e
JOIN (
    SELECT 부서코드, MAX(급여) AS 부서_최대급여
    FROM 직원
    GROUP BY 부서코드
) t                         -- 🚨 별칭(t) 필수! 없으면 에러
  ON e.부서코드 = t.부서코드;
```

> **왜 별칭이 필수인가?** FROM 절의 테이블은 이름으로 참조해야 하는데, 서브쿼리 자체는 이름이 없다. 별칭이 없으면 메인 쿼리에서 이 가상 테이블을 참조할 방법이 없어서 에러가 발생한다. Oracle 은 생략 가능하지만 표준을 위해 항상 붙이는 습관을 들이는 게 좋다.

---

## C. 중첩 서브쿼리 — WHERE / HAVING 절

> **메인 쿼리의 데이터를 조건으로 필터링할 때 사용.**

```sql
SELECT 직원명, 급여
FROM 직원
WHERE 급여 > (SELECT AVG(급여) FROM 직원);   -- 평균보다 큰 사람만
```

---

---

# ② 중첩 서브쿼리 완전 해부

## A. 반환 형태에 따른 분류

> 서브쿼리가 반환하는 결과 크기에 따라 쓸 수 있는 **연산자가 달라진다.**

### 단일행 서브쿼리 (Single-Row)

결과가 **무조건 1건**. 일반 비교 연산자 사용 가능.

|사용 가능 연산자|`=` `>` `<` `>=` `<=` `<>`|
|---|---|

```sql
SELECT 직원명 FROM 직원
WHERE 부서코드 = (SELECT 부서코드 FROM 부서 WHERE 부서명 = '개발부');
--              ↑ 결과가 1건이므로 = 사용 가능
```

---

### 다중행 서브쿼리 (Multi-Row)

결과가 **2건 이상**. `{text}=` 를 쓰면 에러 발생!

|사용 가능 연산자|`IN` `ANY` `ALL` `EXISTS`|
|---|---|

```sql
SELECT 부서명 FROM 부서
WHERE 부서코드 IN (SELECT 부서코드 FROM 직원 WHERE 급여 >= 100000000);
--              ↑ 결과가 여러 건이므로 IN 사용 (= 쓰면 ORA-01427 에러!)
```

> **단일행 연산자 vs 다중행 연산자 호환성:** 단일행 결과에 `IN` 같은 다중행 연산자를 써도 문제없다. 
> 하지만 다중행 결과에 `{text}=` 같은 단일행 연산자를 쓰면 **에러 발생.** → 애매하면 `IN` 을 쓰는 게 안전하다.

---

### EXISTS / NOT EXISTS — 존재 여부만 따진다

> **EXISTS 는 서브쿼리 결과에서 행이 하나라도 존재하면 TRUE, 없으면 FALSE.** 값이 아니라 **존재 여부(TRUE/FALSE)** 만 따진다.

```sql
-- "주문한 적 있는 고객만 출력"
SELECT 고객명 FROM 고객 c
WHERE EXISTS (
    SELECT 1 FROM 주문 o WHERE o.고객ID = c.고객ID
);

-- "한 번도 주문 안 한 고객만 출력"
SELECT 고객명 FROM 고객 c
WHERE NOT EXISTS (
    SELECT 1 FROM 주문 o WHERE o.고객ID = c.고객ID
);
```

---

### SELECT 1 이 뭐야? — 가짜 값의 정체

> EXISTS 는 서브쿼리 결과의 **값이 뭔지 전혀 관심 없다.** "행이 존재하는가?" 만 체크하기 때문에 SELECT 절에 뭘 써도 동작은 동일하다.

```sql
-- 아래 4개는 완전히 동일하게 동작한다
WHERE EXISTS (SELECT 1    FROM 주문 WHERE 고객ID = c.고객ID)
WHERE EXISTS (SELECT 'X'  FROM 주문 WHERE 고객ID = c.고객ID)
WHERE EXISTS (SELECT NULL FROM 주문 WHERE 고객ID = c.고객ID)
WHERE EXISTS (SELECT *    FROM 주문 WHERE 고객ID = c.고객ID)
```

```
EXISTS 의 관심사:
"조건에 맞는 행이 1개라도 있나?" → TRUE
"조건에 맞는 행이 하나도 없나?" → FALSE

SELECT 뒤에 뭘 쓰든 그 값은 보지도 않음
→ 어차피 안 볼 거, 가장 가벼운 숫자 1 을 쓰는 게 관례
```

|SELECT 절|EXISTS 동작|이유|
|---|:-:|---|
|`SELECT *`|동일|모든 컬럼 가져오지만 값 안 봄|
|`SELECT 1`|동일|숫자 1 만 반환, 가장 가벼움 ✅|
|`SELECT 'X'`|동일|문자 X 반환, 값 안 봄|
|`SELECT NULL`|동일|NULL 반환해도 존재 여부만 체크|

> **왜 `SELECT 1` 을 쓰는가?** `SELECT *` 은 모든 컬럼을 가져오는 신호라 옵티마이저가 불필요한 작업을 할 수도 있다. 
> `SELECT 1` 은 "나는 값에 관심 없고 행 존재 여부만 본다" 는 의도를 명확히 전달하는 관례적 작성법이다. 현대 DB 에서는 성능 차이가 없지만 **가독성과 의도 전달** 면에서 `SELECT 1` 을 선호한다.

---

### EXISTS 해석하는 법 — SELECT 1 은 무시하고 WHERE 조건만 읽어라

```
EXISTS  (SELECT 1 FROM 테이블 WHERE 조건)
→ "테이블에서 조건에 맞는 행이 있으면"

NOT EXISTS (SELECT 1 FROM 테이블 WHERE 조건)
→ "테이블에서 조건에 맞는 행이 없으면"
```

```sql
-- 단계별로 읽는 법
SELECT 고객명 FROM 고객 c        -- ① 고객 테이블에서 고객명을 뽑는데
WHERE EXISTS (                   -- ② 존재하면 출력해라
    SELECT 1 FROM 주문 o         -- ③ SELECT 1 은 무시. 주문 테이블에서
    WHERE o.고객ID = c.고객ID    -- ④ 이 고객의 주문 기록이 있는지 확인
);
-- → "주문 기록이 있는 고객만 출력해라"

SELECT 고객명 FROM 고객 c
WHERE NOT EXISTS (               -- "존재하지 않으면 출력해라"
    SELECT 1 FROM 주문 o         -- SELECT 1 무시
    WHERE o.고객ID = c.고객ID    -- 이 고객의 주문 기록이 없는지 확인
);
-- → "한 번도 주문 안 한 고객만 출력해라"
```

```sql
핵심 공식:
SELECT 1 은 항상 무시 → WHERE 뒤 조건만 해석
EXISTS     = "~가 있으면"
NOT EXISTS = "~가 없으면"
```

---

### EXISTS vs IN — 언제 뭘 쓸까?

```sql
-- IN: 서브쿼리 결과 집합과 값 비교
WHERE 고객ID IN (SELECT 고객ID FROM 주문)

-- EXISTS: 연관 서브쿼리로 행 존재 여부 확인
WHERE EXISTS (SELECT 1 FROM 주문 WHERE 주문.고객ID = 고객.고객ID)
```

|구분|`IN`|`EXISTS`|
|---|---|---|
|비교 방식|값 비교 (집합에 포함 여부)|존재 여부 (TRUE/FALSE)|
|NULL 처리|NULL 있으면 UNKNOWN 함정|NULL 무관 (존재만 체크)|
|적합한 상황|서브쿼리 결과가 작을 때|서브쿼리 결과가 클 때|
|연관 서브쿼리|가능하지만 비효율|주로 연관 서브쿼리와 사용|

---

### 다중 컬럼 서브쿼리 (Multi-Column)

결과가 **여러 컬럼 세트.** 컬럼 쌍을 묶어서 비교할 때 사용.

```sql
-- (부서코드, 급여) 두 쌍이 완벽히 일치하는 사람만 필터링
SELECT 직원명, 부서코드, 급여
FROM 직원
WHERE (부서코드, 급여) IN (
    SELECT 부서코드, MAX(급여) FROM 직원 GROUP BY 부서코드
);
```

> ⚠️ **SQL Server(MSSQL) 에서는 이 문법 지원 안 함.** MSSQL 에서는 `EXISTS` 또는 인라인 뷰 + `JOIN` 으로 우회해야 한다.

---

---

## B. 연관성에 따른 분류

### 비연관 서브쿼리 (Uncorrelated) — 독립 실행

서브쿼리가 메인 쿼리와 전혀 무관하게 **혼자서 먼저 한 번만 실행.**

```sql
SELECT 직원명 FROM 직원
WHERE 급여 > (SELECT AVG(급여) FROM 직원);
--            ↑ 메인 쿼리와 상관없이 독립적으로 실행 가능
```

```
실행 순서:
① (SELECT AVG(급여) FROM 직원) 딱 1번 실행 → 결과: 5000
② WHERE 급여 > 5000 으로 필터링
```

---

### 연관 서브쿼리 (Correlated) — 메인 쿼리 행마다 반복 실행

서브쿼리가 **메인 쿼리의 컬럼 값을 참조**해야만 실행 가능.

```sql
SELECT e1.직원명, e1.부서코드, e1.급여
FROM 직원 e1
WHERE e1.급여 > (
    SELECT AVG(e2.급여)
    FROM 직원 e2
    WHERE e2.부서코드 = e1.부서코드   -- 메인 쿼리의 e1.부서코드 를 참조!
);
```

```
실행 순서:
① 메인 쿼리에서 행 1개 읽음 (e1.부서코드 = 100)
② 서브쿼리 실행: WHERE 부서코드 = 100 의 AVG 계산
③ 비교 후 통과/탈락 결정
④ 메인 쿼리에서 행 1개 읽음 (e1.부서코드 = 200)
⑤ 서브쿼리 실행: WHERE 부서코드 = 200 의 AVG 계산
⑥ ... (행 수만큼 반복) → 데이터 많으면 느림!
```

|구분|실행 횟수|속도|
|---|---|---|
|비연관|1번|빠름|
|연관|메인 쿼리 행 수만큼|느림|

---

---

# ③ IN 서브쿼리 완전 정복 — SQLD 킬러 함정 ⭐️

## IN 의 핵심 작동 원리

> **`A IN (SELECT B ...)` = "A 의 값이 B 결과 집합에 포함되는가?"** 다른 컬럼은 절대 자동 비교되지 않는다.

```
잘못된 이해 ❌:
"같은 행에서 COL1 과 COL2 를 비교하는 것"

올바른 이해 ✅:
"COL2 의 값 하나하나를 서브쿼리 결과 집합에 대입해서 포함 여부 확인"
```

---

## SQLD 기출 완전 분석 — SQLD_13 문제

```sql
SELECT COUNT(DISTINCT COL1)
FROM SQLD_13
WHERE COL2 IN (SELECT COL1 FROM SQLD_13);
```

### SQLD_13 테이블

|COL1|COL2|
|---|---|
|1|1|
|1|3|
|2|2|
|3|NULL|

---

### STEP 1 — 서브쿼리 먼저 실행

```sql
SELECT COL1 FROM SQLD_13
```

```
COL1 결과 집합: {1, 1, 2, 3}
IN 에서는 집합처럼 중복 제거 → {1, 2, 3}
```

---

### STEP 2 — WHERE 조건을 행 단위로 대입

> **"COL2 의 값이 {1, 2, 3} 에 포함되는가?"** 를 각 행마다 확인.

|행|COL1|COL2|COL2 IN {1,2,3}?|통과?|
|---|---|---|:-:|:-:|
|1|1|1|1 ∈ {1,2,3}|✅|
|2|1|3|3 ∈ {1,2,3}|✅|
|3|2|2|2 ∈ {1,2,3}|✅|
|4|3|NULL|NULL ∈ {1,2,3}|❌ (NULL 은 항상 UNKNOWN)|

---

### STEP 3 — COUNT(DISTINCT COL1) 계산

```
통과한 행: 행1(COL1=1), 행2(COL1=1), 행3(COL1=2)

DISTINCT COL1 → {1, 2}
COUNT({1, 2}) = 2
```

> **정답: 2**

---

### 왜 COL1=3 인 행(행4)이 제외되었는가?

```
행4: COL1=3, COL2=NULL

WHERE COL2 IN (서브쿼리) 를 평가할 때:
NULL IN {1, 2, 3} = UNKNOWN (NULL 은 = 비교 자체가 불가능)

UNKNOWN 은 TRUE 가 아니므로 → WHERE 조건 탈락!
```

**핵심 착각 포인트:**

```
❌ 잘못된 생각:
"행4 는 COL1=3 이고, 서브쿼리 결과에 3 이 있으니까 통과되겠지?"

✅ 실제 동작:
IN 은 오직 COL2(왼쪽) 값만 집합과 비교한다.
COL1 이 뭔지는 전혀 관계없다.
행4 의 COL2=NULL 이 집합에 포함되는지만 본다.
→ NULL 은 집합 비교 불가 → 탈락
```

---

## 다중 컬럼 IN — 가장 많이 착각하는 부분 ⭐️

>단일 컬럼 IN 은 OR, 다중 컬럼 IN 은 쌍 내부가 AND 다.

```sql
❌ 잘못된 이해:
(COL1, COL2) IN ((1000, 2000))
= COL1=1000 OR COL1=2000 OR COL2=1000 OR COL2=2000

✅ 올바른 이해:
(COL1, COL2) IN ((1000, 2000))
= COL1=1000 AND COL2=2000
```

|COL1|COL2|통과?|이유|
|---|---|---|---|
|1000|2000|✅|COL1=1000 AND COL2=2000 동시 만족|
|1000|9999|❌|COL2 가 2000 이 아님|
|9999|2000|❌|COL1 이 1000 이 아님|
|3000|4000|❌|둘 다 불일치|

```sql
-- 쌍이 여러 개: 쌍 내부는 AND, 쌍끼리는 OR
WHERE (COL1, COL2) IN ((1000, 2000), (3000, 4000))
= (COL1=1000 AND COL2=2000) OR (COL1=3000 AND COL2=4000)
```

|구문|동작 방식|
|---|---|
|`COL1 IN (1000, 2000)`|COL1=1000 **OR** COL1=2000|
|`(COL1, COL2) IN ((1000, 2000))`|COL1=1000 **AND** COL2=2000|
|`(COL1, COL2) IN ((1000, 2000), (3000, 4000))`|(COL1=1000 AND COL2=2000) **OR** (COL1=3000 AND COL2=4000)|

```text
암기 공식:
괄호 한 쌍 (값1, 값2) = AND 로 묶인 하나의 세트
쌍과 쌍 사이           = OR
```

---
## IN / NOT IN 과 NULL — SQLD 단골 함정

```text
IN     =  OR 의 축약
NOT IN =  NOT(OR)  →  드 모르간 법칙으로 AND 로 변환

NOT(A OR B) = NOT A AND NOT B   ← 드 모르간 법칙
```

```sql
-- IN → OR
WHERE COL1 IN ('a', 'b')
= COL1='a' OR COL1='b'

-- NOT IN → NOT(OR) → AND
WHERE COL1 NOT IN ('a', 'b')
= NOT(COL1='a' OR COL1='b')
= COL1!='a' AND COL1!='b'
```

## NULL 이 있을 때 각각 어떻게 되나

### IN (OR) — NULL 행만 못 찾음

```sql
WHERE COL1 IN ('a', NULL, 'b')
= COL1='a' OR COL1=NULL OR COL1='b'
                  ↑
            항상 UNKNOWN (= 로 NULL 비교 불가)

OR 는 하나라도 TRUE 면 통과
→ 'a', 'b' 행은 정상 출력
→ COL1=NULL 인 행만 탈락
```

---

### NOT IN (AND) — 리스트에 NULL 하나라도 있으면 전체 0건 ⚠️

```sql
WHERE COL1 NOT IN ('a', NULL, 'b')
= COL1!='a' AND COL1!=NULL AND COL1!='b'
                    ↑
              항상 UNKNOWN

AND 는 하나라도 UNKNOWN 이면 전체 UNKNOWN
→ 모든 행 탈락 → 결과 0건
```

---
## 다중 컬럼 NOT IN — 드 모르간 2단계 적용

```sql
(COL1, COL2) IN ((a,b), (c,d))
= (COL1=a AND COL2=b) OR (COL1=c AND COL2=d)
--  쌍 내부 AND          쌍끼리 OR
```

### 여기에 NOT 씌우면:

```sql
NOT( (COL1=a AND COL2=b) OR (COL1=c AND COL2=d) )

1단계: OR → AND
= NOT(COL1=a AND COL2=b) AND NOT(COL1=c AND COL2=d)

2단계: 안쪽 AND → OR
= (COL1!=a OR COL2!=b) AND (COL1!=c OR COL2!=d)
```

---
## SQLD IN 문제 풀이 공식

```text
① 서브쿼리를 먼저 계산해서 결과 집합을 구한다
         ↓
② "왼쪽 컬럼(COL2) 의 값이 집합에 있는가?" 로 바꿔 읽는다
         ↓
③ 각 행의 왼쪽 컬럼 값을 집합에 대입한다
         ↓
④ NULL 이 있으면 무조건 UNKNOWN → 탈락
         ↓
⑤ 통과한 행들로 최종 집계
```

>**"왼쪽 컬럼이 주어다."** 오른쪽 서브쿼리는 집합만 제공하고, 비교는 항상 왼쪽 컬럼이 한다. 

---

# ④ 자주 하는 실수

## ORA-01427: single-row subquery returns more than one row

```sql
-- ❌ 에러: 서브쿼리가 여러 건 반환하는데 = 를 사용
WHERE 부서코드 = (SELECT 부서코드 FROM 직원 WHERE 급여 > 5000)

-- ✅ 해결: IN 으로 변경
WHERE 부서코드 IN (SELECT 부서코드 FROM 직원 WHERE 급여 > 5000)
```

## 인라인 뷰 별칭 누락

```sql
-- ❌ 에러: 별칭 없음 (PostgreSQL · MySQL · MSSQL 에서 에러)
SELECT * FROM (SELECT 부서코드, MAX(급여) FROM 직원 GROUP BY 부서코드)

-- ✅ 해결: 별칭 추가
SELECT * FROM (SELECT 부서코드, MAX(급여) FROM 직원 GROUP BY 부서코드) t
```

## NOT IN 에서 NULL 함정

```sql
-- ❌ 위험: 서브쿼리 결과에 NULL 이 있으면 결과 0건
WHERE 부서코드 NOT IN (SELECT 부서코드 FROM 직원)

-- ✅ 안전: NULL 제거 후 비교
WHERE 부서코드 NOT IN (SELECT 부서코드 FROM 직원 WHERE 부서코드 IS NOT NULL)
```

---
