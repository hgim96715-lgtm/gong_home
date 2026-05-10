---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_Standard_JOIN]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 각 사용자에 대해 가입일과 구매자로서 주문한 횟수를 찾는 솔루션을 작성하세요 `2019`.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Users` 테이블
	    - `user_id`
	    - `join_date`
	- `Orders` 테이블
		- `buyer_id`
		- `order_date`
2. **조건 분석:**
	- 주문이 없는 사용자도 같이 나와야 한다. `LEFT JOIN`을 사용
3. **사용할 문법:**
    - `LEFT JOIN`
    - `GROUP BY`
    - PostgreSQL의 `FILTER`
    - MySQL의 `CASE WHEN`
    - PostgreSQL: `EXTRACT(YEAR FROM 날짜)`
    - MySQL: `YEAR(날짜)`
---
## 정답 쿼리 (Solution)

```sql
-- PostgrSQL
SELECT
    u.user_id AS buyer_id,
    u.join_date,
    COUNT(o.order_id) FILTER (
        WHERE EXTRACT(YEAR FROM o.order_date) = 2019
    ) AS orders_in_2019
FROM Users u
LEFT JOIN Orders o
    ON u.user_id = o.buyer_id
GROUP BY
    u.user_id,
    u.join_date;
```

```sql
--MySQL
SELECT
    u.user_id AS buyer_id,
    u.join_date,
    SUM(YEAR(o.order_date) = 2019) AS orders_in_2019
FROM Users u
LEFT JOIN Orders o
    ON u.user_id = o.buyer_id
GROUP BY u.user_id, u.join_date;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT
    u.user_id AS buyer_id,
    u.join_date,
    SUM(
        CASE
            WHEN YEAR(o.order_date) = 2019 THEN 1
            ELSE 0
        END
    ) AS orders_in_2019
FROM Users u
LEFT JOIN Orders o
    ON u.user_id = o.buyer_id
GROUP BY u.user_id, u.join_date;
```
