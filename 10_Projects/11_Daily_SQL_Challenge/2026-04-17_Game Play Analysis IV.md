---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Date_Functions]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 코드를 작성하기 전, 어떻게 풀지 말로 정리해봅니다. (가장 중요!)

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	- `Activity` 테이블
		- `player_id`
		- `event_date`
2. **조건 분석:**
   - 각 `player_id`별로 **최초 로그인 날짜 (MIN(event_date))** 를 구하기
   - 그 날짜의 **다음날 ( +1 day )에 로그인했는지 확인**
1. **사용할 문법:**
   - `MIN() OVER (PARTITION BY ...)`
   - `INTERVAL` / `DATE_ADD()`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- PostgreSQL
WITH pre_check AS(
    SELECT player_id,
           event_date,
           MIN(event_date) OVER(PARTITION BY player_id) AS pre
    FROM Activity
)
SELECT
ROUND(
    COUNT(CASE 
        WHEN event_date = pre + INTERVAL '1 day' 
        THEN player_id 
    END)::NUMERIC
    /
    COUNT(DISTINCT player_id)
, 2) AS fraction
FROM pre_check;
```

```sql
-- MYSQL
WITH pre_check AS(
    SELECT player_id,
           event_date,
           MIN(event_date) OVER(PARTITION BY player_id) AS pre
    FROM Activity
)
SELECT
ROUND(
    COUNT(DISTINCT CASE 
        WHEN event_date = DATE_ADD(pre, INTERVAL 1 DAY) 
        THEN player_id 
    END) * 1.0
    /
    COUNT(DISTINCT player_id)
, 2) AS fraction
FROM pre_check;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
	- 처음에 `LAG()` 사용 → “이전 로그인” 기준으로 잘못 접근
    
-  **기억해야 할 내용/꿀팁:**
	- 그룹별 첫 값 구할 때는 `{sql}MIN(event_date) OVER (PARTITION BY player_id)`

---
## 다른  풀이가 있다면?

```sql
WITH first_login AS (
    SELECT player_id, MIN(event_date) AS first_date
    FROM Activity
    GROUP BY player_id
)
SELECT
ROUND(
    COUNT(DISTINCT a.player_id) * 1.0
    /
    COUNT(DISTINCT f.player_id)
, 2) AS fraction
FROM first_login f
LEFT JOIN Activity a
    ON f.player_id = a.player_id
    AND a.event_date = f.first_date + INTERVAL '1 day';
```
