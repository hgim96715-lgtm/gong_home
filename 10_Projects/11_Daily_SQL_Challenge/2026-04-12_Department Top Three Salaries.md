---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
source: LeetCode
difficulty:
  - Hard
---
##  해결 전략 (Code Before Think)

> 각 부서별로 **상위 3개의 고유 연봉** 안에 드는 직원들을 조회하는 문제

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	-   `Employee` 테이블  
	    → 직원 이름(`name`), 연봉(`salary`), 부서 ID(`departmentId`)
	- `Department` 테이블  
	    → 부서 이름(`name`)
2. **조건 분석:**
   - `DENSE_RANK()` 사용
   - 각 부서별로 나누어야 함 (`PARTITION BY`)
1. **사용할 문법:**
   - `DENSE_RANK() OVER (PARTITION BY ... ORDER BY ...)`
   - `WHERE` : 상위 3개 필터링
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH three AS(

SELECT d.name AS Department,
       e.name AS Employee,
       e.salary AS Salary,
       DENSE_RANK() OVER(PARTITION BY d.name ORDER BY e.salary DESC) AS rnk

FROM Employee e
JOIN Department d ON e.departmentId = d.id

)

SELECT Department, Employee, Salary
FROM three
WHERE rnk <= 3;
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
