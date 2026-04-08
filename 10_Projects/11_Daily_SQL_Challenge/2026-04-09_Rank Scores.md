---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> `Scores` 테이블이 주어졌을 때, 각 점수(`score`)에 대해 순위를 매기세요.  
> 순위는 점수가 높은 순서대로 부여하며, 같은 점수는 같은 순위를 가집니다.  
> 또한, 순위는 연속적으로 이어져야 하며 중간에 건너뛰지 않습니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `Scores` 테이블
1. **조건 분석:**
   - 순위는 건너뛰지 않음 → `DENSE_RANK()` 사용
1. **사용할 문법:**
   - `DENSE_RANK() OVER (ORDER BY score DESC)`
   - (PostgreSQL) `::FLOAT` 형변환
   - (MySQL) `CAST(score AS FLOAT)`
   - MySQL에서는 `rank`가 예약어이므로 → **백틱(`) 처리 필요**
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요. Postgres SQL
SELECT score::FLOAT,
DENSE_RANK()OVER(ORDER BY score DESC) AS rank
FROM Scores
```

```sql
-- MYSQL
SELECT CAST(score AS FLOAT) AS score,
DENSE_RANK()OVER(ORDER BY score DESC) AS `rank`
FROM Scores
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - MySQL에서 `rank`를 그대로 alias로 사용 → **예약어 충돌로 syntax error 발생**
-  **새로 알게 된 함수/꿀팁:**
	- `rank`는 MySQL에서 **예약어(reserved keyword)** 라서백틱으로 감싸야 안전

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
