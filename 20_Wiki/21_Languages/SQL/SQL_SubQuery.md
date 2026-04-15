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
# SQL_SubQuery

> **쿼리 안의 쿼리. 괄호 `()` 안에서 먼저 실행되어 메인 쿼리를 보조하는 하위 쿼리.**

---

## 위치에 따른 3가지 분류

|위치|이름|특징|
|---|---|---|
|`SELECT` 절|스칼라 서브쿼리|컬럼처럼 동작. **반드시 1행 1열** 반환|
|`FROM` 절|인라인 뷰|가상 테이블처럼 동작. **별칭 필수**|
|`WHERE` / `HAVING` 절|중첩 서브쿼리|조건 필터링. 결과 형태에 따라 연산자 달라짐|

---

## 스칼라 서브쿼리 — SELECT 절

반드시 **1행 1열(단일 값)** 만 반환. 컬럼 자리가 올 수 있는 모든 곳에 사용 가능.

```sql
-- SELECT 절: 평균 급여를 옆에 나란히
SELECT
    직원명,
    급여,
    (SELECT AVG(급여) FROM 직원) AS 전체평균급여
FROM 직원;

-- WHERE 절: 동적 기준값으로 필터링
SELECT 직원명 FROM 직원
WHERE 급여 > (SELECT AVG(급여) FROM 직원);

-- UPDATE SET 절: 계산된 값으로 일괄 업데이트
UPDATE 부서
SET 최고급여 = (
    SELECT MAX(급여) FROM 직원
    WHERE 직원.부서코드 = 부서.부서코드
)
WHERE 부서코드 = 100;
```

> SELECT 절에서만 쓰는 게 아니다. 값이 올 수 있는 자리라면 어디든 가능.

---

## 인라인 뷰 — FROM 절

가상 테이블처럼 동작. **별칭이 반드시 필요**하다.

```sql
SELECT e.직원명, t.부서_최대급여
FROM 직원 e
JOIN (
    SELECT 부서코드, MAX(급여) AS 부서_최대급여
    FROM 직원
    GROUP BY 부서코드
) t                          -- 별칭 필수
  ON e.부서코드 = t.부서코드;
```

> 별칭이 없으면 메인 쿼리에서 이 가상 테이블을 참조할 방법이 없어서 에러.

### 인라인 뷰는 JOIN 없이도 쓴다

```sql
-- Window 함수 결과를 WHERE 로 필터링할 때
-- (Window 함수는 WHERE 절에 직접 쓸 수 없어서 이 패턴 필수)
SELECT developer, platform, sales
FROM (
    SELECT
        c.name AS developer,
        p.name AS platform,
        SUM(sales) AS sales,
        RANK() OVER (
            PARTITION BY c.name
            ORDER BY SUM(sales) DESC
        ) AS rnk
    FROM games g
    JOIN companies c ON g.developer_id = c.company_id
    JOIN platforms p ON g.platform_id  = p.platform_id
    GROUP BY c.name, p.name
) t
WHERE rnk = 1;   -- 바깥에서 Window 함수 결과로 필터링
```

```sql
-- 집계 후 다시 집계
SELECT COUNT(*) AS month_count
FROM (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS monthly_sales
    FROM orders
    GROUP BY 1
) t
WHERE monthly_sales >= 1000000;
```

### 인라인 뷰 vs CTE

```sql
-- 인라인 뷰
SELECT * FROM (SELECT ..., RANK() OVER (...) AS rnk FROM ...) t
WHERE rnk = 1;

-- CTE (가독성 더 좋음)
WITH ranked AS (
    SELECT ..., RANK() OVER (...) AS rnk FROM ...
)
SELECT * FROM ranked WHERE rnk = 1;
```

결과 동일. CTE는 가독성, 인라인 뷰는 구버전 DB 호환성.

---

## 중첩 서브쿼리 — WHERE / HAVING 절

반환 행 수에 따라 쓸 수 있는 연산자가 달라진다.

### 단일행 → `{text}=` `>` `<` `>=` `<=` `<>`

```sql
SELECT 직원명 FROM 직원
WHERE 부서코드 = (SELECT 부서코드 FROM 부서 WHERE 부서명 = '개발부');
-- 서브쿼리 결과가 반드시 1건이어야 함. 여러 건이면 에러.
```

### 다중행 → `IN` `ANY` `ALL` `EXISTS`

```sql
SELECT 부서명 FROM 부서
WHERE 부서코드 IN (SELECT 부서코드 FROM 직원 WHERE 급여 >= 10000000);
-- 결과가 여러 건 → IN 사용. = 쓰면 에러 발생.
```

> 단일행 연산자(`{text}=`)를 다중행 결과에 쓰면 에러. 애매하면 `IN`이 안전.

---

## IN / ANY / ALL 완전 정리 ⭐️

### IN — 집합에 포함되는가

```sql
WHERE id IN ('001', '002', '003')
-- id = '001' OR id = '002' OR id = '003' 과 동일

WHERE id IN (SELECT exhibition_id FROM raw_exhibitions WHERE is_active = TRUE)
```

### ANY — 하나라도 만족하면 TRUE

```sql
-- = ANY : IN 과 완전히 동일
WHERE id = ANY(ARRAY['001', '002', '003'])
WHERE id = ANY(SELECT exhibition_id FROM ...)   -- 서브쿼리도 가능

-- < ANY : 서브쿼리 결과 중 최솟값보다 작으면 TRUE
WHERE price < ANY(SELECT price FROM exhibitions WHERE location = '서울')
-- → 서울 전시 중 가장 싼 것보다 저렴한 전시

-- > ANY : 서브쿼리 결과 중 최댓값보다 크면 TRUE
WHERE price > ANY(SELECT price FROM exhibitions WHERE location = '서울')
-- → 서울 전시 중 가장 비싼 것보다 비싼 전시
```

### ALL — 모든 값과 비교해서 전부 만족해야 TRUE

```sql
-- != ALL : 리스트의 모든 값과 다름 → NOT IN 과 동일 (NULL 안전)
WHERE id != ALL(ARRAY['001', '002', '003'])
WHERE id != ALL(%s)   -- psycopg2: 리스트 그대로 전달 가능

-- > ALL : 서브쿼리 결과 중 최댓값보다 크면 TRUE
WHERE price > ALL(SELECT price FROM exhibitions WHERE location = '서울')
-- → 서울 전시 중 가장 비싼 것보다도 비싼 전시

-- < ALL : 서브쿼리 결과 중 최솟값보다 작으면 TRUE
WHERE price < ALL(SELECT price FROM exhibitions WHERE location = '서울')
-- → 서울 전시 중 가장 싼 것보다도 저렴한 전시
```

### ANY / ALL 동등 표현

| 구문                  | 동등 표현                 |
| ------------------- | --------------------- |
| `{text}= ANY(서브쿼리)` | `IN (서브쿼리)`           |
| `!= ALL(배열/서브쿼리)`   | `NOT IN (...)`        |
| `> ALL(서브쿼리)`       | `> (SELECT MAX(...))` |
| `< ALL(서브쿼리)`       | `< (SELECT MIN(...))` |
| `> ANY(서브쿼리)`       | `> (SELECT MIN(...))` |
| `< ANY(서브쿼리)`       | `< (SELECT MAX(...))` |

```
직관적으로 외우는 법:
  ALL  → "모두를 이겨야" → 가장 센 것(MAX) 기준
  ANY  → "하나라도 이기면" → 가장 약한 것(MIN) 기준
```

### NOT IN 대신 `!= ALL` 을 쓰는 이유 — NULL 안전성

```sql
-- NOT IN: 리스트에 NULL 하나라도 있으면 전체 결과 0건
WHERE id NOT IN ('001', NULL, '003')
-- = id != '001' AND id != NULL AND id != '003'
--                   ↑ 항상 UNKNOWN → 전체 AND가 UNKNOWN → 0건

-- != ALL: NULL 있어도 정상 작동
WHERE id != ALL(ARRAY['001', NULL, '003'])
-- 내부적으로 NULL을 무시하고 비교
```

```python
# psycopg2에서 != ALL 사용
active_ids = ['001', '002', '003']

# NOT IN 방식 — 1개일 때 ('001') 문법 오류 위험
cur.execute("WHERE id NOT IN %s", (tuple(active_ids),))

# != ALL 방식 — 개수 무관, 리스트 그대로 전달
cur.execute("WHERE id != ALL(%s)", (active_ids,))
```

---

## EXISTS / NOT EXISTS — 존재 여부만 확인

EXISTS는 서브쿼리 결과의 **값이 뭔지 관심 없음.** "행이 존재하는가?" 만 체크.

```sql
-- 주문한 적 있는 고객만
SELECT 고객명 FROM 고객 c
WHERE EXISTS (
    SELECT 1 FROM 주문 o WHERE o.고객ID = c.고객ID
);

-- 한 번도 주문 안 한 고객만
SELECT 고객명 FROM 고객 c
WHERE NOT EXISTS (
    SELECT 1 FROM 주문 o WHERE o.고객ID = c.고객ID
);
```

### SELECT 1 이 뭐야?

```sql
-- 아래 4개는 완전히 동일하게 동작
WHERE EXISTS (SELECT 1    FROM 주문 WHERE 고객ID = c.고객ID)
WHERE EXISTS (SELECT 'X'  FROM 주문 WHERE 고객ID = c.고객ID)
WHERE EXISTS (SELECT NULL FROM 주문 WHERE 고객ID = c.고객ID)
WHERE EXISTS (SELECT *    FROM 주문 WHERE 고객ID = c.고객ID)
```

EXISTS는 값을 보지 않으므로 SELECT 뒤에 뭘 써도 동일. → 의도를 명확히 전달하는 가장 가벼운 `SELECT 1`이 관례.

### EXISTS 읽는 법

```
EXISTS  (SELECT 1 FROM 테이블 WHERE 조건)
→ "테이블에서 조건에 맞는 행이 있으면"

→ SELECT 1 은 무시하고 WHERE 조건만 읽으면 됨
```

### EXISTS vs IN

|구분|`IN`|`EXISTS`|
|---|---|---|
|비교 방식|값 비교 (집합 포함 여부)|존재 여부 (TRUE/FALSE)|
|NULL 처리|NULL 있으면 UNKNOWN 함정|NULL 무관|
|적합한 상황|서브쿼리 결과 작을 때|서브쿼리 결과 클 때|
|연관 서브쿼리|가능하지만 비효율|주로 연관 서브쿼리와 사용|

---

## IN 핵심 동작 원리

**"왼쪽 컬럼이 주어다."** 오른쪽 서브쿼리는 집합만 제공.

```sql
WHERE COL2 IN (SELECT COL1 FROM T)
-- "COL2의 값이 서브쿼리 결과 집합에 있는가?" 만 본다
-- COL1이 뭔지는 비교에 전혀 영향 없음
```

### 다중 컬럼 IN — AND / OR 논리

```sql
-- 단일 컬럼: OR
WHERE COL1 IN ('a', 'b')
= COL1='a' OR COL1='b'

-- 다중 컬럼: 쌍 내부는 AND, 쌍끼리는 OR
WHERE (COL1, COL2) IN (('a', 1), ('b', 2))
= (COL1='a' AND COL2=1) OR (COL1='b' AND COL2=2)
```

> ⚠️ SQL Server(MSSQL)에서는 다중 컬럼 IN 지원 안 함 → EXISTS나 JOIN으로 우회.

---

## NULL 함정 정리 ⭐️

```
IN → NULL 행만 못 찾음 (나머지 정상)
NOT IN → 리스트에 NULL 하나라도 있으면 전체 0건!
```

**이유 — 드 모르간 법칙으로 변환**

```sql
-- NOT IN → NOT(OR) → AND 로 변환됨
WHERE COL1 NOT IN ('a', NULL, 'b')
= NOT(COL1='a' OR COL1=NULL OR COL1='b')
= COL1!='a' AND COL1!=NULL AND COL1!='b'
--              ↑ 항상 UNKNOWN
-- AND는 하나라도 UNKNOWN이면 전체 UNKNOWN → 0건!
```

**해결법**

```sql
-- NOT IN 사용 시 반드시 NULL 제거
WHERE 부서코드 NOT IN (
    SELECT 부서코드 FROM 직원 WHERE 부서코드 IS NOT NULL
)

-- 또는 != ALL 로 교체 (NULL 안전)
WHERE 부서코드 != ALL(SELECT 부서코드 FROM 직원)
```

| |NULL 있을 때 영향|
|---|---|
|`IN`|NULL 행만 못 찾음. 나머지 정상|
|`NOT IN`|리스트에 NULL 하나라도 있으면 **전체 0건**|
|`!= ALL`|NULL 있어도 정상 작동 ← 권장|

---

## 연관 vs 비연관 서브쿼리

### 비연관 (Uncorrelated) — 독립 실행, 1번만

```sql
SELECT 직원명 FROM 직원
WHERE 급여 > (SELECT AVG(급여) FROM 직원);
-- 서브쿼리가 메인 쿼리와 무관 → 1번만 실행
```

### 연관 (Correlated) — 메인 쿼리 행마다 반복

```sql
SELECT e1.직원명, e1.급여
FROM 직원 e1
WHERE e1.급여 > (
    SELECT AVG(e2.급여)
    FROM 직원 e2
    WHERE e2.부서코드 = e1.부서코드   -- 메인 쿼리 컬럼 참조
);
-- e1 행 하나마다 서브쿼리 1번 실행 → 데이터 많으면 느림
```

|구분|실행 횟수|속도|
|---|---|---|
|비연관|1번|빠름|
|연관|메인 쿼리 행 수만큼|느림|

---

## 자주 하는 실수

```sql
-- 다중행에 = 사용 → 에러
WHERE 부서코드 = (SELECT 부서코드 FROM 직원 WHERE 급여 > 5000)
-- ✅ 수정
WHERE 부서코드 IN (SELECT 부서코드 FROM 직원 WHERE 급여 > 5000)

-- 인라인 뷰 별칭 누락 → 에러
SELECT * FROM (SELECT 부서코드, MAX(급여) FROM 직원 GROUP BY 부서코드)
-- ✅ 수정
SELECT * FROM (SELECT 부서코드, MAX(급여) FROM 직원 GROUP BY 부서코드) t

-- NOT IN + NULL 함정 → 결과 0건
WHERE 부서코드 NOT IN (SELECT 부서코드 FROM 직원)
-- ✅ 수정
WHERE 부서코드 NOT IN (SELECT 부서코드 FROM 직원 WHERE 부서코드 IS NOT NULL)
-- 또는
WHERE 부서코드 != ALL(SELECT 부서코드 FROM 직원)
```

---

---