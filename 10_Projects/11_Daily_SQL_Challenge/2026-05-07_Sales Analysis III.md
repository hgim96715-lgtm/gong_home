---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_SubQuery]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> 2019년 1분기, 즉 `2019-01-01`부터 `2019-03-31`까지의 기간에만 판매된 제품을 조회하는 문제입니다.

결과 테이블을 **어떤 순서** 로든 반환합니다 .

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Product` 테이블
	    - `product_id`
	    - `product_name`
	- `Sales` 테이블
		- `product_id`
		- `sale_date`
2. **조건 분석:**
    - 가장 빠른 판매일이 2019-01-01 이상이어야 한다.
    - 가장 늦은 판매일이 2019-03-31 이하여야 한다.
3. **사용할 문법:**
    - `MIN(sale_date)`
    - `MAX(sale_date)`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT p.product_id,p.product_name

FROM Product p

JOIN Sales s ON p.product_id=s.product_id

GROUP BY p.product_id,p.product_name

HAVING MIN(sale_date) >= '2019-01-01' AND MAX(sale_date)<='2019-03-31'
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에는 `WHERE`절에 다음처럼 조건을 걸면 된다고 생각했다.
    - 하지만 이렇게 하면 **1분기에 한 번이라도 판매된 상품**이 조회된다.
-  **새로 알게 된 함수/꿀팁:**
	- `MIN()`과 `MAX()`를 이용하면 특정 그룹의 전체 범위를 확인할 수 있다.

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT DISTINCT
    p.product_id,
    p.product_name
FROM Product p
JOIN Sales s
    ON p.product_id = s.product_id
WHERE s.sale_date BETWEEN '2019-01-01' AND '2019-03-31'
  AND NOT EXISTS (
      SELECT 1
      FROM Sales s2
      WHERE s2.product_id = p.product_id
        AND s2.sale_date NOT BETWEEN '2019-01-01' AND '2019-03-31'
  );
```
