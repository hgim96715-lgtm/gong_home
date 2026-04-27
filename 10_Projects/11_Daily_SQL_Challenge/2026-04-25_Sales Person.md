---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_Understanding_NULL]]"
  - "[[SQL_SubQuery]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> 영업사원 정보, 회사 정보, 주문 정보가 주어졌을 때,  회사 이름이 **'RED'** 인 회사와 **주문 거래를 한 적이 없는 영업사원의 이름**을 조회하세요.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `SalesPerson` 테이블
	    - `name`
	    - `sales_id`
	- `Orders` 테이블
		- `sales_id`
		- `com_id`
	- `Company` 테이블
		- `name`
		- `com_id`
2. **조건 분석:**
    - `Orders`와 `Company`를 조인해서 `c.name = 'RED'`인 주문을 찾는다.
3. **사용할 문법:**
   - `LEFT JOIN` : 전체 영업사원 기준으로 RED 거래 여부 확인
   - `IS NULL` : RED 거래 기록이 없는 사람만 필터링하기
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH c_o AS (
    SELECT o.com_id, o.sales_id
    FROM Orders o
    JOIN Company c 
        ON o.com_id = c.com_id
    WHERE c.name = 'RED'
)

SELECT s.name AS name
FROM SalesPerson s
LEFT JOIN c_o o1 
    ON s.sales_id = o1.sales_id
WHERE o1.sales_id IS NULL;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에는 `WHERE o.com_id = 1`처럼 회사 ID를 직접 지정했다.
    - 하지만 문제에서는 회사 이름이 `'RED'`인 회사를 찾아야 하므로, `com_id`가 항상 1이라고 가정하면 안 된다.
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT s.name
FROM SalesPerson s
WHERE s.sales_id != ALL (
    SELECT o.sales_id
    FROM Orders o
    JOIN Company c
        ON o.com_id = c.com_id
    WHERE c.name = 'RED'
);
```
