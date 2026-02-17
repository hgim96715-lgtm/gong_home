---
aliases:
  - 제약조건
  - PK
  - FK
  - NOT NULL
  - DEFAULT
tags:
  - SQL
related:
  - "[[SQL_DDL_Create]]"
  - "[[00_SQL_HomePage]]"
---
## 개념 한 줄 요약

**"이상한 데이터가 들어오지 못하게 막는 '클럽의 문지기(Bouncer)' 같은 규칙들"**

---
## `PRIMARY KEY` (기본키, PK) 🔑

- **의미:** **"주민등록번호"**. 테이블에서 데이터를 식별하는 유일한 ID.
- **특징:** 중복될 수 없고(Unique), 비어있을 수 없습니다(Not Null).
- **용도:** `user_id`, `order_id`, `event_uuid` 등.

----
##  `NOT NULL` (필수 입력) 🚫

- **의미:** **"빈칸 금지"**. 무조건 값이 있어야 함.
- **용도:** 회원가입 시 이름, 비밀번호 등.

---
##  `DEFAULT` (기본값) 

- **의미:** **"안 적으면 이거 써"**. 값을 입력 안 했을 때 자동으로 들어가는 값.
- **용도:** `created_at` (입력 안 하면 `NOW()`로 현재 시간 자동 저장).

---
## `UNIQUE` (중복 금지) 

- **의미:** **"너랑 똑같은 애 있으면 안 돼"**.
- **용도:** 이메일 주소, 전화번호 (한 사람이 두 번 가입 막기).

---
## 실전 테이블

```sql
CREATE TABLE products (
    -- [PK] 상품 ID는 유일해야 하고 비면 안 됨
    product_id INT PRIMARY KEY,
    
    -- [NOT NULL] 상품명은 필수
    product_name VARCHAR(100) NOT NULL,
    
    -- [DEFAULT] 가격 안 적으면 0원 처리
    price INT DEFAULT 0,
    
    -- [UNIQUE] 바코드는 전 세계에서 유일해야 함
    barcode VARCHAR(50) UNIQUE,
    
    -- [DEFAULT] 등록일은 자동으로 현재 시간
    created_at TIMESTAMP DEFAULT NOW()
);
```