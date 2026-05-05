---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 각 제품이 판매되기 시작한 **첫 해** 에 발생한 모든 판매량을 찾는 방법을 작성하세요 .
>  각 항목에 대해 표에서 `product_id`가장 먼저 `year`나타나는 항목을 찾으십시오 `Sales`.
>   해당 연도의 해당 제품에 대한 **모든** 판매 내역을 반환합니다 .
>   **product_id** , **first_year** , **quantity,** price **열** 을 포함하는 테이블을 반환하세요 .  
>   결과는 순서에 상관없이 반환해야 합니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Sales` 테이블
	    - `product_id`
	    - `quantity`
	    - `year`
	    - `price`
2. **조건 분석:**
    - 각 `product_id`별로 가장 빠른 `year`를 찾아야 한다
    - 같은 제품이 같은 첫 해에 여러 번 판매될 수 있으므로, 첫 해에 해당하는 모든 행을 반환
3. **사용할 문법:**
    - `RANK() OVER(PARTITION BY product_id ORDER BY year)`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH sales_cnt AS (
    SELECT
        product_id,
        year,
        RANK() OVER (
            PARTITION BY product_id
            ORDER BY year
        ) AS year_nct,
        quantity,
        price
    FROM Sales
)

SELECT
    product_id,
    year AS first_year,
    quantity,
    price
FROM sales_cnt
WHERE year_nct = 1;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에는 `ROW_NUMBER()`를 사용하려고 했다.
    - 하지만 `ROW_NUMBER()`는 같은 연도라도 행마다 `1, 2, 3...`처럼 서로 다른 번호를 붙인다.
    - 첫 해의 판매 기록이 여러 개일 수 있으므로, 첫 번째 행 하나만 가져오면 오답이 된다.
-  **새로 알게 된 함수/꿀팁:**
	- `RANK()`는 같은 정렬 기준 값이면 같은 순위를 부여한다.
	- “최솟값에 해당하는 모든 행”을 구하는 문제면 `RANK()` 또는 `MIN()`을 고려한다.
---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT 
    product_id,
    year AS first_year,
    quantity,
    price
FROM Sales
WHERE (product_id, year) IN (
    SELECT 
        product_id,
        MIN(year)
    FROM Sales
    GROUP BY product_id
);
```
