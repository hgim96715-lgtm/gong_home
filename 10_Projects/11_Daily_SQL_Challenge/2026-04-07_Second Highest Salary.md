---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Aggregate_GROUP_BY#⭐️ 예외 — 결과 자체가 NULL 이 되는 경우]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 두 번째로 높은 **중복 제거된 급여(distinct salary)** 를 구하고, 없으면 NULL 반환

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	- `Employee` 테이블
		- `salary` 
2. **조건 분석:**
   - 만약 2등이 없다면 → **NULL 반환**
1. **사용할 문법:**
   - `DENSE_RANK()` → 중복을 하나의 순위로 처리
   - `ORDER BY salary DESC` → 높은 값부터 정렬
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH rnk_emp AS (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM Employee
)
SELECT MAX(salary) AS SecondHighestSalary
FROM rnk_emp
WHERE rnk = 2;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
	- 처음에 문제 요구사항은 “값이 없으면 NULL 반환”인데,집계 함수 없이 결과를 그대로 반환하려고 해서 **NULL 처리 로직이 빠짐**
    
-  **새로 알게 된 함수/꿀팁:**
	- `MAX()` 같은 **집계 함수는 결과가 없을 때 NULL을 반환함**
	- **“조건 결과가 없을 수 있으면, 반드시 집계로 감싸서 NULL을 강제하자”**

>[[SQL_Aggregate_GROUP_BY#⭐️ 예외 — 결과 자체가 NULL 이 되는 경우]] 참조 
---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
