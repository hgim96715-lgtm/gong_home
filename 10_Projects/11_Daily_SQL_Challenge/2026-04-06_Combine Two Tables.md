---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Standard_JOIN]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> Person 테이블의 모든 사람을 기준으로, Address 정보가 있으면 같이 출력하고 없으면 NULL을 출력해야 한다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	- `Person` 테이블
		- `lastName`
		- `firstName`
		- `personId`
	-  `Address` 테이블
		- `city`
		- `state`
2. **조건 분석:**
   - Address에 없는 경우도 보여야 하므로 →LEFT JOIN
1. **사용할 문법:**
   - `LEFT JOIN` → 기준 테이블(Person)은 전부 유지
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT p.firstName AS firstName,
p.lastName AS lastName,
a.city AS city,
a.state AS state
FROM Person p
LEFT JOIN Address a ON p.personId=a.personId
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
