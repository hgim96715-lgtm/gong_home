---
aliases:
  - dbt 모델
  - ref
  - source
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Project_Structure]]"
  - "[[DBT_Materializations]]"
  - "[[DBT_Sources]]"
---

# DBT_Models — SQL 변환 모델

## 한 줄 요약

```
dbt 모델 = SELECT 문만 쓰는 .sql 파일
dbt 가 자동으로 테이블 또는 뷰로 변환
```

---

---

# ① dbt 모델이란

```
일반 SQL:
  CREATE TABLE fct_orders AS SELECT ...
  → 테이블 생성 / 삭제 / DDL 직접 관리

dbt 모델:
  SELECT 문만 작성
  → dbt 가 자동으로 CREATE TABLE / VIEW 처리
  → 개발자는 변환 로직에만 집중

핵심:
  .sql 파일 = 1개 모델
  파일명 = 테이블/뷰 이름
  SELECT 만 쓰면 됨
```

---

---

# ② 모델 파일 기본 구조

```sql
-- models/staging/stg_orders.sql

SELECT
    order_id,
    customer_id,
    order_date::DATE        AS order_date,
    amount::NUMERIC(10, 2)  AS amount,
    status
FROM {{ source('raw', 'orders') }}   -- 원본 테이블 참조
WHERE status != 'cancelled'
```

```
파일명: stg_orders.sql
→ DB 에 stg_orders 라는 테이블/뷰 생성

{{ source() }}  → 원본 DB 테이블 참조
{{ ref() }}     → 다른 dbt 모델 참조
```

---

---

# ③ source() vs ref() ⭐️

```
source()  → DB 에 이미 존재하는 원본 테이블 참조
            sources.yml 에 정의한 테이블만 사용 가능
            staging 모델에서 주로 사용

ref()     → 다른 dbt 모델 참조
            자동으로 의존성(DAG) 관리
            실행 순서 자동 결정
```

```sql
-- source() 사용 — 원본 테이블
SELECT *
FROM {{ source('jaffle_shop', 'orders') }}
--               ↑소스명        ↑테이블명
-- sources.yml 에 정의된 것만 사용 가능

-- ref() 사용 — 다른 모델
SELECT *
FROM {{ ref('stg_orders') }}
--          ↑ 모델 파일명 (확장자 없이)
```

## 모델 레이어 흐름

```
원본 DB
  ↓ source('raw', 'orders')
staging 모델 (stg_orders)
  ↓ ref('stg_orders')
intermediate 모델 (int_order_summary)
  ↓ ref('int_order_summary')
mart 모델 (fct_orders)
```

```sql
-- models/marts/fct_orders.sql
SELECT
    o.order_id,
    o.order_date,
    o.amount,
    c.customer_name
FROM {{ ref('stg_orders') }} o           -- 다른 모델 참조
JOIN {{ ref('stg_customers') }} c
    ON o.customer_id = c.customer_id
```

---

---

# ④ config 블록 — 모델별 설정

```sql
-- 파일 맨 위에 config 블록으로 이 모델만 설정 변경
{{ config(
    materialized='table',      -- view / table / incremental / ephemeral
    schema='analytics',        -- 저장할 스키마
    tags=['daily', 'finance'], -- 태그 (선택 실행 시 사용)
    alias='my_orders'          -- 실제 테이블명 변경
) }}

SELECT ...
```

```
우선순위:
  config 블록 > dbt_project.yml > 기본값
  개별 모델 설정이 전체 설정보다 우선
```

---

---

# ⑤ dbt run — 모델 실행 ⭐️

```
dbt run = SELECT 문을 실제 DB 에 테이블/뷰로 만드는 명령어

처음에 헷갈리는 것:
  SQL 파일을 작성했다고 자동으로 DB 에 생기지 않음
  반드시 dbt run 을 실행해야 DB 에 반영됨
```

```bash
dbt run                          # 전체 모델 실행
dbt run --select stg_orders      # 특정 모델만
dbt run --select staging.*       # staging 폴더 전체
dbt run --select +fct_orders     # fct_orders + 의존 모델 전부
dbt run --select fct_orders+     # fct_orders + 하위 모델 전부
```

```
실행 로그:
  01:23:45  1 of 3 START sql view model staging.stg_orders ......... [RUN]
  01:23:46  1 of 3 OK created sql view model staging.stg_orders ..... [OK in 0.80s]
  01:23:46  2 of 3 START sql view model staging.stg_customers ....... [RUN]
  ...

START/OK 가 나오면 → DB 에 모델이 생성됨
ephemeral 모델은 START/OK 안 나옴 (CTE 로 삽입되기 때문)
```

---

---

# ⑥ 모델 네이밍 컨벤션

```
stg_  → staging 모델 (원본 정제)
int_  → intermediate 모델 (중간 변환)
fct_  → fact 모델 (이벤트 데이터)
dim_  → dimension 모델 (속성 데이터)
```

```
stg_orders.sql
stg_customers.sql
int_order_items.sql
fct_orders.sql
dim_customers.sql
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|모델 작성 후 DB 에 없음|`dbt run` 안 함|`dbt run` 실행|
|`ref()` 에서 모델 못 찾음|파일명 오타|파일명과 `ref('이름')` 일치 확인|
|`source()` 에서 테이블 못 찾음|sources.yml 미정의|sources.yml 에 먼저 등록|
|SELECT * 사용|컬럼 변경 시 문제|컬럼 명시적으로 나열|