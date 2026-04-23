---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_UNION]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 친구가 가장 많은 사람과 가장 많은 친구 수를 찾는 풀이 과정을 작성하세요.
> 테스트 케이스는 한 사람만 가장 많은 친구를 갖도록 생성됩니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `RequestAccepted` 테이블
	    - `requester_id`
	    - `accepter_id`
	    - `accept_date`
2. **조건 분석:**
    - 합쳐진 ID 리스트에서 각 ID별로 빈도수를 계산
    - 빈도수가 가장 높은 최상위 1개의 레코드만 추출
3. **사용할 문법:**
   - `GROUP BY` & `COUNT`
   - `UNION ALL`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH together_id AS (
    SELECT requester_id AS id FROM RequestAccepted
    UNION ALL
    SELECT accepter_id AS id FROM RequestAccepted
)

SELECT 
    id, 
    COUNT(id) AS num
FROM together_id
GROUP BY id
ORDER BY num DESC
LIMIT 1;
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
