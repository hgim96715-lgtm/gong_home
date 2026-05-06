---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_Standard_JOIN]]"
  - "[[SQL_Numeric_Functions]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> 각 프로젝트별 모든 직원의 **평균 경력 연수를** **소수점 둘째 자리까지 반올림하여** 출력하는 SQL 쿼리를 작성하세요

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Project` 테이블
	    - `employee_id`
	    - `project_id`
	- `Employee` 테이블
		- `employee_id`
		- `experience_years`
2. **조건 분석:**
    - 프로젝트별 평균을 구해야 하므로 `project_id` 기준으로 그룹화
    - 직원의 경력 연수는 `Employee` 테이블에 있으므로 `Project`와 `Employee`를 `employee_id` 기준으로 `JOIN`
    - 평균 경력 연수는 소수점 둘째 자리까지 반올림해야 하므로 `ROUND()`를 사용한다.
3. **사용할 문법:**
    - `JOIN`
    - PostgreSQL 형 변환 `::numeric`
    - MySQL 실수 연산을 위한 `* 1.0`
---
## 정답 쿼리 (Solution)

```sql
-- Postgre SQL
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT  
p.project_id,  
ROUND((SUM(e.experience_years)::numeric / COUNT(*)), 2) AS average_years  
FROM Project p  
JOIN Employee e  
ON p.employee_id = e.employee_id  
GROUP BY p.project_id  
ORDER BY p.project_id;
```

```sql
--My SQL
-- My SQL
SELECT 
    p.project_id,
    ROUND((SUM(e.experience_years) * 1.0 / COUNT(*)), 2) AS average_years
FROM Project p
JOIN Employee e 
    ON p.employee_id = e.employee_id
GROUP BY p.project_id
ORDER BY p.project_id;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**
	- 평균은 직접 `SUM() / COUNT()`로 구할 수도 있지만, SQL에서는 `AVG()` 함수를 사용하면 더 간단하게 구할 수 있다.

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
-- PostgreSQL / MySQL 공통으로 더 간단한 풀이
SELECT 
    p.project_id,
    ROUND(AVG(e.experience_years), 2) AS average_years
FROM Project p
JOIN Employee e 
    ON p.employee_id = e.employee_id
GROUP BY p.project_id;
```
