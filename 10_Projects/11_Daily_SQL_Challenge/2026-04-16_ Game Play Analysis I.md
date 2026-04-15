---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

>  각 플레이어의 **첫 로그인 날짜**를 구하는 쿼리를 작성하세요. 결과 테이블은 **아무 순서로 반환해도 됩니다.**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
	-  `Activity` 테이블
		- `player_id`
		- `event_date`
2. **조건 분석:**
    - "first login date" = **가장 처음 로그인한 날짜**
    - 같은 `player_id` 내에서 **최소(event_date)** 를 구하기
3. **사용할 문법:**
   - `GROUP BY player_id`
   - `MIN(event_date)`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT player_id,
	MIN(event_date) AS first_login
FROM Activity
GROUP BY player_id
ORDER BY player_id
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

