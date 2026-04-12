---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_DML_CRUD#DELETE + JOIN / USING — 다른 테이블 참조해서 삭제 ⭐️]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> 중복된 email을 가진 행들이 있는 데 그 중에서 가장 작은 id를 제외한 나머지행 삭제하라

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `Person` 테이블
1. **조건 분석:**
   - 같은 email을 가진 행들을 비교해야 함
   - `id`가 더 큰 값 = 중복 → 삭제 대상
1. **사용할 문법:**
   - `DELETE ... USING` (PostgreSQL)
   - `DELETE` + `JOIN` (MySQL)
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- MYSQL
DELETE p1
FROM Person p1
JOIN Person p2 
ON p1.email = p2.email 
AND p1.id > p2.id;
```

```sql
--PostgreSQL
DELETE FROM Person p1
USING Person p2
WHERE p1.email = p2.email
AND p1.id > p2.id;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
DELETE FROM Person
WHERE id IN (
    SELECT id FROM (
        SELECT id,
               ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
        FROM Person
    ) t
    WHERE rn > 1
);
```
