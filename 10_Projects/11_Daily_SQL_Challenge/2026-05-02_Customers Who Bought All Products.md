---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_SubQuery]]"
  - "[[SQL_DISTINCT_vs_GROUP_BY]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> `Product` 테이블에 있는 **모든 제품을 구매한 고객의 ID**를 출력하는 문제입니다.  
> 결과 테이블은 **어떤 순서로 반환해도 됩니다.**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Customer` 테이블
	    - `customer_id`
	    - `product_key`
	- `Product` 테이블
		- `product_key`
2. **조건 분석:**
    - 고객이 구매한 서로 다른 상품 개수 = Product 테이블의 전체 상품 개수
3. **사용할 문법:**
    - `GROUP BY` 
    - `COUNT(DISTINCT 컬럼명)`
    - `HAVING`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT c.customer_id
FROM Customer c
GROUP BY c.customer_id
HAVING COUNT(DISTINCT c.product_key) = (
    SELECT COUNT(*)
    FROM Product
);
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에는 `JOIN`으로 해결하려고 했다.
    - 하지만 이 쿼리는 고객이 구매한 상품 목록을 보여줄 뿐, **모든 상품을 구매했는지**는 판단하지 못한다.
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
