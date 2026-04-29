---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_CASE_WHEN]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 연속된 두 학생씩 자리를 서로 바꾸되, 학생 수가 홀수면 마지막 학생은 그대로 두고 `id` 순서대로 출력하는 문제입니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Seat` 테이블
	    - `id`
	    - `student`
2. **조건 분석:**
    - 홀수 `id`는 다음 자리와 바꿔야 하므로 `id + 1`
    - 짝수 `id`는 이전 자리와 바꿔야 하므로 `id - 1`
3. **사용할 문법:**
    - `CASE WHEN` 
    -  `WITH`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH change_id AS (
    SELECT
        id,
        student,
        CASE
            WHEN id = (SELECT MAX(id) FROM Seat) AND id % 2 = 1 THEN id
            WHEN id % 2 = 1 THEN id + 1
            ELSE id - 1
        END AS id_change
    FROM Seat
)

SELECT
    id_change AS id,
    student
FROM change_id
ORDER BY id_change;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - `COUNT(*)`는 단독 조건으로 사용할 수 없고, `id = COUNT(*)`처럼 비교 대상이 필요
    - `CASE`문에서는 더 구체적인 조건인 “마지막 홀수 id”를 먼저 검사
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT
    CASE
        WHEN id = (SELECT MAX(id) FROM Seat) AND id % 2 = 1 THEN id
        WHEN id % 2 = 1 THEN id + 1
        ELSE id - 1
    END AS id,
    student
FROM Seat
ORDER BY id;
```
