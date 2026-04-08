---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Top_N_Query]]"
  - "[[SQL_Stored_Function]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> **중복을 제거한 급여(distinct salary)** 중에서   **N번째로 높은 값**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `Employee`  테이블
	   - 컬럼: `salary`
1. **조건 분석:**
   - 
3. **사용할 문법:**
   - DENSE_RANK() OVER (ORDER BY salary DESC)
   - WITH (CTE)
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요. PostgreSQL
CREATE OR REPLACE FUNCTION NthHighestSalary(N INT)
RETURNS TABLE (salary_out INT) AS $$
BEGIN
  RETURN QUERY (
      WITH rnk_salary AS (
        SELECT DISTINCT salary,
               DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
        FROM Employee
      )
      SELECT MAX(rnk_salary.salary)
      FROM rnk_salary
      WHERE rnk = N
  );
END;
$$ LANGUAGE plpgsql;

```

```sql
-- MYSQL
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  RETURN(
    WITH rnk_salary AS (
      SELECT DISTINCT salary,
             DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
      FROM Employee
    )
    SELECT salary
    FROM rnk_salary
    WHERE rnk = N
  );
END;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**
	- MY SQL ->`RETURN QUERY` 없음 , `RETURN (SELECT ...)` 형태 써야 함
	- PostgreSQL ->`RETURN QUERY` , `RETURNS TABLE`

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
