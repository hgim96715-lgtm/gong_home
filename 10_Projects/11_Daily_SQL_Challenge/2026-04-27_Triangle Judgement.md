---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_CASE_WHEN]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> `Triangle` 테이블의 각 행에 있는 세 선분 `x, y, z`가 삼각형을 만들 수 있는지 `Yes` 또는 `No`로 판단하세요.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Triangle` 테이블
	    - `x`
	    - `y`
	    - `z`
2. **조건 분석:**
    - 세 변이 삼각형이 되려면 두 변의 합이 나머지 한 변보다 커야 한다.
3. **사용할 문법:**
   - `CASE WHEN` : 조건에 따라 `Yes`, `No` 출력
   - `AND` : 삼각형 조건 3개를 모두 만족하는지 확인
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT x, y, z,
    CASE
        WHEN x + y > z
         AND x + z > y
         AND y + z > x
        THEN 'Yes'
        ELSE 'No'
    END AS triangle
FROM Triangle;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
	- 처음에는 조건을 여러 개의 `WHEN`으로 나누어 작성했다.
	- 삼각형은 조건 3개 중 하나만 만족하면 되는 것이 아니라, **3개를 모두 만족**해야 한다
	- 여러 `WHEN`이 아니라 하나의 `WHEN` 안에서 `AND`로 연결해야 한다.
-  **새로 알게 된 함수/꿀팁:**
	- 삼각형 조건은 `두 변의 합 > 나머지 한 변`이다

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
