---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> `Orders` 테이블이 주어지고, 각 주문(`order_number`)은 하나의 고객(`customer_number`)에 속한다.
> **가장 많은 주문을 한 고객의 `customer_number`를 찾는 문제**
> 한 고객이 여러 번 주문할 수 있음 **주문 수가 가장 많은 고객은 유일하게 1명** 

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Orders` 테이블
	    - `customer_number`별 주문 개수
2. **조건 분석:**
     - 고객별로 주문 횟수를 세야 함 → `GROUP BY`
     - 가장 많이 주문한 고객 찾기 → `ORDER BY COUNT DESC`
     - 1명만 반환 → `LIMIT 1`
3. **사용할 문법:**
     -  `GROUP BY`
     - `COUNT()`
     - `ORDER BY`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH cnt_num AS (
    SELECT 
        customer_number,
        COUNT(*) AS cnt
    FROM Orders
    GROUP BY customer_number
)

SELECT customer_number
FROM cnt_num
ORDER BY cnt DESC
LIMIT 1;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
