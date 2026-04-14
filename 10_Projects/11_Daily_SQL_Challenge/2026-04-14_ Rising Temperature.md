---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Date_Functions]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> 특정 날짜의 기온이 **전날(어제)**보다 높은 경우의 `id`를 찾는 문제

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	- `Weather` 테이블
		- 각 날짜(`recordDate`)별 온도(`temperature`)
2. **조건 분석:**
	-  단순히 이전 행이 아니라 **"정확히 하루 전 날짜"와 비교해야 함**
3. **사용할 문법:**
   - 윈도우 함수 `LAG()`
   - PostgreSQL: `INTERVAL '1 day'`
   - MySQL: `DATEDIFF()`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- PostgreSQL
WITH last_temp AS(
	SELECT id,temperature,recordDate,
	LAG(temperature,1) OVER(ORDER BY recordDate) AS lag_temp,
	LAG(recordDate) OVER (ORDER BY recordDate) AS prev_date
	FROM Weather
)

SELECT id
FROM last_temp
WHERE temperature>lag_temp AND recordDate=prev_date + INTERVAL '1 day'
```

```sql
-- MYSQL
WITH last_temp AS(
	SELECT id,temperature,recordDate,
	LAG(temperature,1) OVER(ORDER BY recordDate) AS lag_temp,
	LAG(recordDate) OVER (ORDER BY recordDate) AS prev_date
	FROM Weather
)
SELECT id
FROM last_temp
WHERE temperature>lag_temp AND DATEDIFF(recordDate,prev_date) = 1
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에 날짜 차이를 확인하지 않아서 이틀 전 데이터까지 포함되는 오류 발생
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
