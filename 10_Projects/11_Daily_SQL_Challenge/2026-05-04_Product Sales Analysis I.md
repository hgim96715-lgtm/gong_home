---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_Standard_JOIN]]"
  - "[[SQL_Standard_JOIN#USING — 내가 지정한 컬럼만 조인]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> **판매 기록(Sales)에 있는 각 판매 건에 대해 상품명, 판매연도, 가격을 조회해주세요**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Sales` 테이블
	    - `year`
	    - `price`
	    - `product_id`
	- `Product` 테이블
		- `product_name`
		- `product_id`
2. **조건 분석:**
    - 두 테이블은 공통 컬럼인 `product_id`로 연결할 수 있다.
3. **사용할 문법:**
    - `JOIN`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
SELECT 
    p.product_name,
    s.year,
    s.price
FROM Sales s
JOIN Product p 
ON s.product_id = p.product_id;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT 
    product_name,
    year,
    price
FROM Sales
INNER JOIN Product
USING (product_id);
```
