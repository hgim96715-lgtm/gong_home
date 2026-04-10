---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions#① 순위 함수]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 코드를 작성하기 전, 어떻게 풀지 말로 정리해봅니다. (가장 중요!)

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Logs` 테이블
	    - `num` → 연속 여부 판단 대상
	    - `id` → 정렬 기준
2. **조건 분석:**
   - 같은 숫자(`num`)가 **연속해서 3번 이상 등장해야 함**
1. **사용할 문법:**
   - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id)
   - `GROUP BY`
   - `HAVING COUNT(*)`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- MYSQL , PostgresSQL
WITH row_id AS (
    SELECT num,
           ROW_NUMBER() OVER (ORDER BY id) -
           ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS row_num
    FROM Logs
)

SELECT num AS ConsecutiveNums
FROM row_id
GROUP BY num, row_num
HAVING COUNT(*) >= 3;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에 `ROW_NUMBER() OVER (PARTITION BY num ORDER BY id)` 하나만 사용하려고 해서 잘 되지 않음
    - 연속인지 여부는 전혀 판단 못함
-  **새로 알게 된 함수/꿀팁:**
	- `ROW_NUMBER()`를 **두 번 사용해서 차이를 구하면** 연속된 구간을 그룹화할 수 있다

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT DISTINCT num AS ConsecutiveNums
FROM (
    SELECT num,
           LAG(num, 1) OVER (ORDER BY id) AS prev1,
           LAG(num, 2) OVER (ORDER BY id) AS prev2
    FROM Logs
) t
WHERE num = prev1
  AND num = prev2;
```
