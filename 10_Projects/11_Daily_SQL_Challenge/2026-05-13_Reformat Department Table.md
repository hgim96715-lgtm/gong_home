---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_Window_Functions]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

>**부서 ID 열과 월별** 매출액 열이 있는 형태로 표 형식을 변경하십시오 .
>결과 테이블을 **어떤 순서** 로든 반환합니다

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Department` 테이블
	    - `id`
	    - `month`
	    - `revenue`
2. **조건 분석:**
    - `id`를 기준으로 `GROUP BY`
    - 각 월 컬럼은 `CASE WHEN month = 'Jan' THEN revenue END`처럼 특정 월에 해당하는 매출만 가져와야 한다.
3. **사용할 문법:**
    - `CASE WHEN`
    - `SUM()`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- Write your PostgreSQL query statement below

SELECT id,
       SUM(CASE WHEN month = 'Jan' THEN revenue END) AS Jan_Revenue,
       SUM(CASE WHEN month = 'Feb' THEN revenue END) AS Feb_Revenue,
       SUM(CASE WHEN month = 'Mar' THEN revenue END) AS Mar_Revenue,
       SUM(CASE WHEN month = 'Apr' THEN revenue END) AS Apr_Revenue,
       SUM(CASE WHEN month = 'May' THEN revenue END) AS May_Revenue,
       SUM(CASE WHEN month = 'Jun' THEN revenue END) AS Jun_Revenue,
       SUM(CASE WHEN month = 'Jul' THEN revenue END) AS Jul_Revenue,
       SUM(CASE WHEN month = 'Aug' THEN revenue END) AS Aug_Revenue,
       SUM(CASE WHEN month = 'Sep' THEN revenue END) AS Sep_Revenue,
       SUM(CASE WHEN month = 'Oct' THEN revenue END) AS Oct_Revenue,
       SUM(CASE WHEN month = 'Nov' THEN revenue END) AS Nov_Revenue,
       SUM(CASE WHEN month = 'Dec' THEN revenue END) AS Dec_Revenue
FROM Department
GROUP BY id
ORDER BY id;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에는 원본 데이터를 그대로 조회하려고 했다.
    -  `month` 값을 각각의 월별 컬럼으로 바꾸는 **피벗(Pivot)** 문제
-  **새로 알게 된 함수/꿀팁:**
	- SQL에서 행(row)을 열(column) 형태로 바꾸고 싶을 때는 `CASE WHEN`과 집계 함수 `SUM()`을 함께 사용할 수 있다.
---
## 더 나은 풀이가 있다면?

```sql
-- PostgreSQL에서는 FILTER 문법을 사용해서 더 깔끔하게 작성할 수도 있다.
SELECT id,
       SUM(revenue) FILTER (WHERE month = 'Jan') AS Jan_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Feb') AS Feb_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Mar') AS Mar_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Apr') AS Apr_Revenue,
       SUM(revenue) FILTER (WHERE month = 'May') AS May_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Jun') AS Jun_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Jul') AS Jul_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Aug') AS Aug_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Sep') AS Sep_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Oct') AS Oct_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Nov') AS Nov_Revenue,
       SUM(revenue) FILTER (WHERE month = 'Dec') AS Dec_Revenue
FROM Department
GROUP BY id
ORDER BY id;
```
