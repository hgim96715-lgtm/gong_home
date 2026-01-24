---
aliases:
  - CASE WHEN
  - SQL IF문
  - 조건문
  - 조건부 집계
tags:
  - SQL
related:
  - "[[Aggregation_GROUP_BY]]"
  - "[[Pivot_Unpivot]]"
---
## 개념 한 줄 요약

SQL에서 **"만약(IF) ~라면 A, 아니면(ELSE) B"** 라는 논리를 구현하는 문법이야.
엑셀의 `IF` 함수나 프로그래밍의 `if-else` 문과 똑같아.

---
## 왜 필요한가 (Why)

1.  **데이터 범주화 (Categorization):** 점수(85점)를 등급('A학점')으로 바꿀 때.
2.  **데이터 정제 (Cleaning):** `M`, `F`로 된 성별을 `Male`, `Female`로 바꿀 때.
3.  **조건부 집계 (Conditional Aggregation):** (핵심🔥) 전체 매출 중 '전자제품' 매출만 따로 컬럼으로 뽑고 싶을 때.

---
##  Basic Syntax (기본 문법)

```sql
SELECT
    product_name,
    price,
    CASE 
        WHEN price >= 10000 THEN 'High'  -- 조건 1
        WHEN price >= 5000  THEN 'Mid'   -- 조건 2
        ELSE 'Low'                       -- 나머지 전부 (생략 시 NULL)
    END AS price_category                -- ⭐️ 끝맺음(END)과 별칭(AS) 필수!
FROM products;
```

---
## Practical Context (실전 활용: 조건부 집계)

**"데이터 엔지니어는 CASE WHEN을 SUM 안에 넣는다."**
단순히 값을 바꾸는 것을 넘어, **피벗 테이블**처럼 데이터를 옆으로 펼칠 때 사용해.

Q. 날짜별로 '가입자 수'와 '탈퇴자 수'를 한 줄에 보고 싶다면?

```sql
SELECT
    event_date,
    -- status가 'join'인 행만 1로 바꿔서 다 더함
    SUM(CASE WHEN status = 'join' THEN 1 ELSE 0 END) AS join_count,
    -- status가 'leave'인 행만 1로 바꿔서 다 더함
    SUM(CASE WHEN status = 'leave' THEN 1 ELSE 0 END) AS leave_count
FROM user_logs
GROUP BY event_date;
```

> **해석:** 
>  `COUNT`를 쓰면 조건에 상관없이 다 세버리지만, `SUM + CASE WHEN`을 쓰면 **원하는 조건만 콕 집어서 카운팅**할 수 있어.
>   (이 기술이 진짜 중요해!)

---
## 초보자가 자주 착각하는 포인트

- **Q. `ELSE`를 안 쓰면 어떻게 돼?**
    - 조건에 안 맞는 데이터는 자동으로 **`NULL`** 이 돼. 
    - 빈 값이 생기는 걸 원치 않으면 `ELSE 'Unknown'` 처럼 기본값을 꼭 넣어줘.

- **Q. `END`를 자꾸 까먹어.**
    - SQL에서 `CASE`를 열었으면 무조건 `END`로 닫아줘야 해. 괄호를 닫는 것과 같은 이치야.


