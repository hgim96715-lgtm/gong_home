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

> `(actor_id, director_id)`배우가 감독과 최소 세 번 이상 협업한 모든 쌍을 찾는 풀이를 작성하세요 .
> 결과 테이블을 **어떤 순서** 로든 반환합니다 .

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `ActorDirector` 테이블
    - `actor_id`, 
    - `director_id`
2. **조건 분석:**
    - `actor_id`, `director_id` 조합별로 그룹화
    - 그룹별 행 개수가 3개 이상인 경우만 출력
3. **사용할 문법:**
    - `GROUP BY actor_id, director_id`
    - `COUNT(*)`
    - `HAVING COUNT(*) >= 3`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT actor_id, director_id  
FROM ActorDirector  
GROUP BY actor_id, director_id  
HAVING COUNT(*) >= 3;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**
	- `GROUP BY`는 같은 값끼리 묶을 때 사용
	- 두 컬럼을 함께 `GROUP BY`하면 `(actor_id, director_id)` 쌍 단위로 묶인다.

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
