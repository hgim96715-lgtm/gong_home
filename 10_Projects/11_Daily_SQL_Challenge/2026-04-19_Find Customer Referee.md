---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_NULL_Functions]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> 코드를 작성하기 전, 어떻게 풀지 말로 정리해봅니다. (가장 중요!)

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	- `Customer` 테이블
		- `name`
		- `referee_id`
2. **조건 분석:**
	-  `referee_id != 2` 인 경우
	- `referee_id`가 NULL인 경우 (추천인이 없는 경우)
3. **사용할 문법:**
   - `IS NULL` → NULL 체크
   - `WHERE` → 조건 필터링
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT name
FROM Customer
WHERE referee_id !=2 OR referee_id IS NULL
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
