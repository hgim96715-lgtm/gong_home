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

> `MyNumbers` 테이블에는 정수 `num` 값들이 들어 있습니다.  
> 이 테이블에는 같은 숫자가 여러 번 나올 수 있습니다.
> 여기서 **single number**란 테이블에서 **딱 한 번만 등장한 숫자**를 의미합니다.
> 이 문제는 **한 번만 등장한 숫자들 중 가장 큰 숫자**를 찾는 문제입니다.  
> 만약 한 번만 등장한 숫자가 없다면 `null`을 반환해야 합니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `MyNumbers` 테이블 
	    - `num`
2. **조건 분석:**
    -  `COUNT(num) = 1` 인 숫자만 single number이다.
    - single number 중 최댓값을 구해야 한다.
    - 조건에 맞는 숫자가 없으면 `MAX()` 결과가 자동으로 `null`이 된다.
3. **사용할 문법:**
	-  `COUNT()`
	- `MAX()`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH cnt_num AS (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(num) = 1
)

SELECT MAX(num) AS num
FROM cnt_num;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT MAX(num) AS num
FROM (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(*) = 1
) AS single_numbers;
```
