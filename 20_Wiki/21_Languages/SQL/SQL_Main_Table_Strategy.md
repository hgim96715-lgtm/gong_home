---
aliases:
  - 메인 테이블 선정
  - 기준 테이블
  - One Row Per What
  - 쿼리 주인공 찾기
  - Driving Table
tags:
  - SQL
related:
  - "[[SQL_Execution_Order]]"
  - "[[01_SQL_Thinking_Roadmap]]"
  - "[[00_SQL_HomePage]]"
---
## 개념 한 줄 요약

쿼리를 짤 때 `FROM` 절에 가장 먼저 적는 테이블은 **"분석의 기준(Base)"** 이자 **"주인공"** 이다.
나머지 `JOIN`으로 붙는 테이블들은 주인공을 설명해주는 **"액세서리(Details)"** 일 뿐이다.

---
##  The Golden Rule (절대 원칙) 

`FROM` 테이블을 정할 때, 딱 하나의 질문만 던져라.

> **"결과표의 한 줄(1 Row)이 무엇을 의미해야 하는가?"**
> (Data Granularity, 데이터의 결/입자)

* 결과가 **"고객 1명"** 이어야 한다? 👉 `FROM Customers`
* 결과가 **"주문 1건"** 이어야 한다? 👉 `FROM Orders`
* 결과가 **"상품 1개"** 여야 한다? 👉 `FROM Products`

---
##  Practical Steps (3단계 사고법)

헷갈릴 때는 이 순서대로 생각하면 무조건 풀려.

### Step 1. "보고 싶은 결과물" 상상하기

요청사항: *"고객별로 총 얼마 썼는지 뽑아줘. 주문 안 한 고객도 포함해서!"*

### Step 2. "주인공(기준)" 찾기

* **고객(Customer)** 이 주인공인가? **주문(Order)** 이 주인공인가?
* "주문 안 한 고객도 포함" -> 주문이 없어도 고객은 나와야 함.
* ⭐️ **주인공 = 고객 (Customer)**

### Step 3. 배치하기 (Left Join의 법칙)

* **주인공을 `FROM`에 놓는다.**
* 나머지는 `LEFT JOIN`으로 붙인다. (주인공을 살리기 위해)

```sql
SELECT c.name, SUM(o.amount)
FROM customers c       -- 👑 주인공 (기준)
LEFT JOIN orders o     -- 💍 액세서리 (정보 제공)
  ON c.id = o.customer_id
GROUP BY c.name;
```

---
## 4. Why is it important? (잘못 골랐을 때의 비극)

만약 위 상황에서 **주문(Orders)**을 `FROM`에 뒀다면?

```sql
-- ❌ 잘못된 선택
SELECT c.name, SUM(o.amount)
FROM orders o          -- 엥? 주문이 주인공?
LEFT JOIN customers c
  ON o.customer_id = c.id
...
```

- **문제점:** "주문 안 한 고객"은 `orders` 테이블에 아예 없지?
- **결과:** 주문 내역이 없는 신규 회원은 **리포트에서 아예 누락됨.** (상사가 "왜 회원 수가 이거밖에 안 돼?" 하고 화냄)

---
## Advanced Context (1:N 관계에서의 전략)

테이블이 여러 개(`Users` - `Orders` - `Order_Items`)일 때가 진짜 헷갈리지? 
이때는 **"가장 상세한(Deepest) 레벨"** 이 누군지 봐야 해.

**질문 A:** "회원 수 몇 명이야?"

- 주인공: `Users` (1명 = 1줄)
- `FROM Users`

**질문 B:** "총 매출 얼마야?"

- 주인공: `Orders` (1건 = 1줄)
- `FROM Orders`

**질문 C:** "가장 많이 팔린 **상품 옵션**이 뭐야?"

- 주인공: `Order_Items` (주문서 안의 상세 품목)
- `FROM Order_Items`

**💡 핵심 팁:** 
통계를 낼 때(`COUNT`, `SUM`), 내가 `FROM`에 둔 테이블보다 **더 상세한(자식) 테이블**을 `JOIN` 하면 데이터가 뻥튀기(Fan-out)되어서 값이 2배, 3배로 뛸 수 있어.
👉 **가장 하위 레벨(자식) 테이블을 `FROM`에 두는 것이 계산 실수를 줄이는 지름길이야.**

---
## Summary Table (상황별 추천)

|**분석 목적**|**주인공 (FROM)**|**조인 방향**|
|---|---|---|
|**회원 리스트, 회원 등급**|`Users` (부모)|`Users` LEFT JOIN `Orders`|
|**일별 매출, 주문 건수**|`Orders` (자식)|`Orders` LEFT JOIN `Users`|
|**상품별 판매량**|`Order_Items` (손자)|`Order_Items` LEFT JOIN `Products`|
