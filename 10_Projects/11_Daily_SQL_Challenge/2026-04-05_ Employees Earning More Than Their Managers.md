---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Self_Join]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> Employee 테이블에서
> **자기 매니저보다 연봉이 높은 직원의 이름을 찾는 문제**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `Employee` 테이블
	   - `e.name` (직원 이름)
	   - `e.salary` → 직원 급여
	   - `e.managerId` 매니저 id
1. **조건 분석:**
   - 직원과 매니저 연결 
   - self join 
1. **사용할 문법:**
   - `JOIN` (self join 필수)
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT e.name AS Employee
FROM Employee e
JOIN Employee c ON e.managerId = c.id
WHERE e.salary>c.salary
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
	- 처음에 JOIN 방향을 반대로 설정함
	- `e.id = c.managerId` 매니저 → 직원 (오답)
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
