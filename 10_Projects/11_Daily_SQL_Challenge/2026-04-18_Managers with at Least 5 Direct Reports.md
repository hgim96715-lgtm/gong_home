---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Self_Join]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> Employee 테이블에서 **직속 부하(direct reports)가 5명 이상인 매니저의 이름**을 조회하라.
> 각 직원은 `managerId`로 자신의 매니저를 가리킴
> `managerId`가 null이면 매니저가 없는 직원
> 한 매니저 기준으로 **직접 보고하는 직원 수가 5 이상**인 경우만 출력

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `Employee` 테이블
	   - `managerId`는 해당 직원의 매니저 `id`
	   - `id`는 기본 키 (각 직원의 고유 ID)
1. **조건 분석:**
	-  `e1.id = e2.managerId` 관계
2. **사용할 문법:**
   - `COUNT()` : 각 매니저의 부하 직원 수 계산
   - `HAVING` : 그룹 결과에서 조건 필터링 (COUNT ≥ 5)
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT e1.name
FROM Employee e1
JOIN Employee e2 ON e1.id=e2.managerId
GROUP BY e1.id,e1.name
HAVING COUNT(e2.id)>=5
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
