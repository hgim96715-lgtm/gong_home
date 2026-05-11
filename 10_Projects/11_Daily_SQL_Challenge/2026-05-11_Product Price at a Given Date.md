---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_NULL_Functions]]"
  - "[[SQL_Standard_JOIN]]"
  - "[[SQL_Window_Functions]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> 초기에 모든 제품의 가격은 10입니다.  
> 주어진 날짜 `2019-08-16`의 모든 제품 가격을 구하는 풀이 과정을 작성하세요.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Products` 테이블
	    - `product_id`
	    - `new_price`
	    - `change_date`
2. **조건 분석:**
    - `change_date <= '2019-08-16'` 조건을 만족하는 가격 변경만 유효
    - 전체 상품이 결과에 나와야 하므로, 먼저 모든 상품 목록을 만든 뒤 `LEFT JOIN` 해야 한다.
    - 조인 결과가 없으면 `COALESCE` 또는 `IFNULL`로 가격을 `10`으로 처리
3. **사용할 문법:**
    - PostgreSQL: `DISTINCT ON`
    - MySQL: `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)`
    - 공통: `WITH`, `LEFT JOIN`, `COALESCE` / `IFNULL`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- PostgreSQL
WITH latest_price AS (
    SELECT DISTINCT ON (product_id)
           product_id,
           new_price AS price
    FROM Products
    WHERE change_date <= '2019-08-16'
    ORDER BY product_id, change_date DESC
),
all_products AS (
    SELECT DISTINCT product_id
    FROM Products
)
SELECT a.product_id,
       COALESCE(l.price, 10) AS price
FROM all_products a
LEFT JOIN latest_price l
    ON a.product_id = l.product_id;
```

```sql
-- MySQL
WITH ranked AS (
    SELECT product_id,
           new_price,
           ROW_NUMBER() OVER (
               PARTITION BY product_id
               ORDER BY change_date DESC
           ) AS rn
    FROM Products
    WHERE change_date <= '2019-08-16'
),
all_products AS (
    SELECT DISTINCT product_id
    FROM Products
)
SELECT a.product_id,
       IFNULL(r.new_price, 10) AS price
FROM all_products a
LEFT JOIN ranked r
    ON a.product_id = r.product_id
   AND r.rn = 1;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    -  `WHERE change_date <= '2019-08-16'` 조건만 걸면, 해당 날짜 이전 가격이 있는 상품만 남게 된다.
    - 모든 상품 목록을 따로 만든 뒤, 기준일 이전의 최신 가격과 `LEFT JOIN` 해야 한다.
-  **새로 알게 된 함수/꿀팁:**
	- `ROW_NUMBER()`를 사용하면 상품별 최신 데이터를 쉽게 뽑을 수 있다.
	- PostgreSQL에서는 `DISTINCT ON (product_id)`를 사용하면 더 간결하게 최신 행을 뽑을 수 있다.
	- `COALESCE()` 또는 `IFNULL()`은 `NULL` 값을 기본값으로 바꿀 때 사용
---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
