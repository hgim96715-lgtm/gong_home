---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_DISTINCT_vs_GROUP_BY]]"
  - "[[HAVING_vs_WHERE]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> `author_id`와 `viewer_id`가 같다면, **글을 쓴 사람이 자기 자신의 글을 조회했다**는 뜻입니다.
> 문제는 **자신이 작성한 글을 최소 한 번 이상 조회한 저자들의 ID**를 찾는 것입니다.
> 단, 같은 저자가 여러 번 자기 글을 봤을 수도 있으므로 중복을 제거해야 합니다.
> 결과는 `id` 기준으로 오름차순 정렬해야 합니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Views` 테이블
	    - `author_id`
	    - `viewer_id`
2. **조건 분석:**
    - `author_id = viewer_id`인 행만 선택
    - 같은 저자가 여러 번 자기 글을 조회했을 수 있으므로 중복 제거가 필요 `DISTINCT`를 사용
3. **사용할 문법:**
    - `SELECT DISTINCT`
    - `WHERE`
    - `ORDER BY`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT DISTINCT author_id AS id
FROM Views
WHERE author_id = viewer_id
ORDER BY id;
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
