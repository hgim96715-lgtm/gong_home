---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> **학생이 5명 이상인 class만 찾기**  
> 결과는 순서 상관 없음 / class 컬럼만 출력

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `Courses` 테이블
	   - `class`
1. **조건 분석:**
	-  class 기준 그룹화
2. **사용할 문법:**
   - `GROUP BY` → class 기준 그룹화

---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(*)>=5
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(*) >= 5;
```
