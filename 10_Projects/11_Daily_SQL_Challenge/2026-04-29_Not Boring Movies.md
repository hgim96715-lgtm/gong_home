---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Filtering_WHERE]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> `Cinema` 테이블에서 영화 정보를 조회해야 합니다.
> `id`가 **홀수**인 영화만 선택, `description`이 **"boring"이 아닌** 영화만 선택
> 결과는 `rating` 기준으로 **내림차순 정렬**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Cinema` 테이블
	    - `id`
	    - `movie`
	    - `description`
	    - `rating`
2. **조건 분석:**
    - 홀수 ID만 → `id % 2 = 1`
    - 설명이 `"boring"`이 아니어야 함 → `description != 'boring'`
3. **사용할 문법:**
   - `ORDER BY DESC`
   - `!=`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT id, movie, description, rating
FROM Cinema
WHERE id % 2 = 1
  AND description != 'boring'
ORDER BY rating DESC;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT *
FROM Cinema
WHERE id % 2 = 1
  AND description <> 'boring'
ORDER BY rating DESC;
```
