---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Standard_JOIN]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

>고객이 선호하는 배송일이 주문일과 같으면 **즉시 배송 주문이고, 그렇지 않으면** **예약** 배송 주문입니다 .
>고객의 첫 **주문** 은 고객이 주문한 주문 중 가장 이른 날짜의 주문입니다. 고객은 정확히 하나의 첫 주문을 갖도록 보장됩니다.
>**모든 고객의 첫 주문 중 즉시 주문의 비율을 소수점 둘째 자리까지 반올림하여** 구하는 풀이 과정을 작성하세요 .

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Delivery` 테이블
	    - `order_date`
	    - `customer_pref_delivery_date`
	    - `customer_id`
	    - `delivery_id`
2. **조건 분석:**
    - 즉시 배송 조건 :`order_date = customer_pref_delivery_date`
3. **사용할 문법:**
    - `MIN() OVER(PARTITION BY ...)`
    - `COUNT(*) FILTER(WHERE ...)`
    - `ROUND(값, 2)`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
-- PostgreSQL
WITH first_order AS (
    SELECT 
        delivery_id,
        customer_id,
        order_date,
        customer_pref_delivery_date,
        MIN(order_date) OVER (
            PARTITION BY customer_id
        ) AS first_order_date
    FROM Delivery
)
SELECT
    ROUND(
        COUNT(*) FILTER (
            WHERE order_date = customer_pref_delivery_date
        ) * 100.0 / COUNT(*),
        2
    ) AS immediate_percentage
FROM first_order
WHERE order_date = first_order_date;
```

```sql
-- MySQL
WITH ranked AS (
    SELECT
        delivery_id,
        customer_id,
        order_date,
        customer_pref_delivery_date,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date
        ) AS rn
    FROM Delivery
)
SELECT
    ROUND(
        SUM(order_date = customer_pref_delivery_date) * 100.0 / COUNT(*),
        2
    ) AS immediate_percentage
FROM ranked
WHERE rn = 1;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
   - 처음에는 전체 주문을 기준으로 즉시 배송 비율을 계산했다.
   - 하지만 문제에서 요구한 것은 **전체 주문 중 즉시 배송 비율**이 아니라, **고객별 첫 주문 중 즉시 배송 비율**이었다.
   - 비율을 계산할 때 `COUNT / COUNT`만 하면 정수 나눗셈 문제가 생길 수 있으므로 `* 100.0`을 사용해야 한다.
-  **새로 알게 된 함수/꿀팁:**
	- MySQL에서는 조건식이 참이면 `1`, 거짓이면 `0`으로 계산되므로 `SUM(조건식)`을 사용할 수 있다.
	- 고객별 첫 번째 데이터를 찾을 때는 `ROW_NUMBER()`를 사용할 수 있다.

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
SELECT
    ROUND(
        SUM(d.order_date = d.customer_pref_delivery_date) * 100.0 / COUNT(*),
        2
    ) AS immediate_percentage
FROM Delivery d
JOIN (
    SELECT
        customer_id,
        MIN(order_date) AS first_order_date
    FROM Delivery
    GROUP BY customer_id
) f
    ON d.customer_id = f.customer_id
   AND d.order_date = f.first_order_date;
```
