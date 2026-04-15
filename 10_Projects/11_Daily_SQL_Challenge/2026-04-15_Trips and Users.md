---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Standard_JOIN]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
source: LeetCode
difficulty:
  - Hard
---
##  해결 전략 (Code Before Think)

> 취소율(Cancellation Rate)은  **취소된 요청 수 / 전체 요청 수**로 계산한다.  
> 단, **client와 driver 모두 banned = 'No'인 경우만 포함**하며,  
> 특정 날짜 범위(2013-10-01 ~ 2013-10-03) 내에서 하루별로 계산한다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	- `Trips` 테이블
		- `status` (취소 여부 판단)
		- `request_at` (날짜 기준 그룹핑)
		- `client_id`, `driver_id`
	- `Users` 테이블
		- `banned` 여부 확인
		- `users_id`
2. **조건 분석:**
	- client와 driver **둘 다 banned = 'No'**
3. **사용할 문법:**
   - `COUNT(*)` : 전체 건수
   - `JOIN` : client / driver 각각 Users와 연결 (2번 조인)
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- Write your PostgreSQL query statement below
WITH join_two AS (
    SELECT
        t.request_at,
        t.status
    FROM Trips t
    JOIN Users u1 
        ON t.client_id = u1.users_id 
        AND u1.banned = 'No'
    JOIN Users u2 
        ON t.driver_id = u2.users_id 
        AND u2.banned = 'No'
    WHERE t.request_at BETWEEN '2013-10-01' AND '2013-10-03'
)

SELECT
    request_at AS Day,
    ROUND(
        COUNT(*) FILTER (WHERE status LIKE 'cancelled_by_%')::NUMERIC
        / COUNT(*),
        2
    ) AS "Cancellation Rate"
FROM join_two
GROUP BY request_at
ORDER BY request_at;
```

```sql
-- MYSQL
WITH join_two AS (
    SELECT
        t.request_at,
        t.status
    FROM Trips t
    JOIN Users u1 
        ON t.client_id = u1.users_id 
        AND u1.banned = 'No'
    JOIN Users u2 
        ON t.driver_id = u2.users_id 
        AND u2.banned = 'No'
    WHERE t.request_at BETWEEN '2013-10-01' AND '2013-10-03'
)

SELECT
    request_at AS Day,
    ROUND(
       SUM(IF(status != 'completed', 1, 0)) / COUNT(*),
        2
    ) AS "Cancellation Rate"
FROM join_two
GROUP BY request_at
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
    request_at AS Day,
    ROUND(AVG(status != 'completed'), 2) AS "Cancellation Rate"
FROM Trips t
JOIN Users u1 ON t.client_id = u1.users_id AND u1.banned = 'No'
JOIN Users u2 ON t.driver_id = u2.users_id AND u2.banned = 'No'
WHERE t.request_at BETWEEN '2013-10-01' AND '2013-10-03'
GROUP BY request_at;
```
