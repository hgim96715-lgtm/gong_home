---
aliases:
  - materialization
  - dbt
  - table
  - view
  - incremental
  - ephemeral
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Models]]"
  - "[[DBT_Project_Structure]]"
---

# DBT_Materializations — Materialization 종류

## 한 줄 요약

```
Materialization = dbt 가 모델을 DB 에 어떻게 저장할지 결정
view / table / incremental / ephemeral 4가지
```

---

---

# ① 4가지 종류 비교 ⭐️

|종류|DB 저장|실행 시|용도|
|---|---|---|---|
|`view`|뷰 생성|매번 쿼리 실행|기본값 / 가벼운 변환|
|`table`|테이블 생성|전체 재생성|최종 분석 테이블|
|`incremental`|테이블 (증분)|새 데이터만 추가|대용량 이벤트 로그|
|`ephemeral`|저장 안 함|CTE 로 삽입|중간 단계 / 임시|

---

---

# ② view — 기본값

```
SQL 뷰로 생성
데이터를 저장하지 않음
조회할 때마다 SQL 실행
빠르게 만들 수 있음 / 무겁지 않음
```

```sql
{{ config(materialized='view') }}   -- 기본값이라 생략 가능

SELECT
    order_id,
    amount
FROM {{ source('raw', 'orders') }}
```

```
장점: 항상 최신 데이터 / 빠른 배포
단점: 조회마다 계산 → 복잡한 쿼리면 느림
```

---

---

# ③ table — 실제 테이블 생성

```
실제 테이블로 생성
데이터를 물리적으로 저장
dbt run 할 때마다 전체 삭제 후 재생성 (DROP + CREATE)
```

```sql
{{ config(materialized='table') }}

SELECT
    order_id,
    customer_id,
    SUM(amount) AS total_amount
FROM {{ ref('stg_orders') }}
GROUP BY 1, 2
```

```
장점: 조회 빠름 / 복잡한 집계에 적합
단점: 전체 재생성 → 데이터 많으면 오래 걸림
```

---

---

# ④ incremental — 증분 업데이트

```
처음 실행: 전체 데이터로 테이블 생성
이후 실행: 새 데이터만 추가 (기존 테이블 유지)

대용량 이벤트 로그 / 클릭스트림 등에 적합
매번 전체 재생성하면 너무 오래 걸리는 경우 사용
```

```sql
{{ config(
    materialized='incremental',
    unique_key='event_id'        -- 중복 방지 기준 컬럼
) }}

SELECT
    event_id,
    user_id,
    event_type,
    created_at
FROM {{ source('raw', 'events') }}

{% if is_incremental() %}
-- 처음 실행 시 이 조건 없음 → 전체 적재
-- 이후 실행 시 → 마지막 데이터 이후만 가져옴
WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

```
is_incremental():
  처음 실행 → False → WHERE 조건 없음 → 전체 적재
  이후 실행 → True  → WHERE 조건 있음 → 증분 적재

{{ this }} = 현재 모델 자신 (이미 존재하는 테이블)
```

```bash
# 전체 재생성 강제 (증분 무시)
dbt run --select my_model --full-refresh
```

---

---

# ⑤ ephemeral — CTE 로 삽입 ⭐️

```
DB 에 테이블/뷰를 생성하지 않음
이 모델을 참조하는 다운스트림 모델의 SQL 에
CTE (WITH 구문) 형태로 코드가 삽입됨
```

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='ephemeral') }}

SELECT
    order_id,
    amount
FROM {{ source('raw', 'orders') }}
```

```sql
-- models/marts/fct_orders.sql
SELECT *
FROM {{ ref('stg_orders') }}
```

```sql
-- dbt 가 실제로 실행하는 SQL (컴파일 결과)
WITH stg_orders AS (        -- ← stg_orders 가 CTE 로 삽입됨
    SELECT
        order_id,
        amount
    FROM raw.orders
)
SELECT *
FROM stg_orders
```

```
ephemeral 모델의 특징:
  DB 에 독립적인 테이블 / 뷰 생성 안 함
  참조하는 모델 SQL 안에 CTE 로 녹아들어감
  dbt run 로그에 START/OK 출력 안 됨
  중간 단계 / 재사용 로직에 사용

장점: DB 에 불필요한 뷰 만들지 않음 / 깔끔
단점: 직접 조회 불가 / 디버깅 어려움
```

## ephemeral 실행 로그 차이

```
-- 일반 모델 (view / table)
01:23:45  START sql view model staging.stg_orders ....... [RUN]
01:23:46  OK created sql view model staging.stg_orders .. [OK]

-- ephemeral 모델
(아무것도 안 나옴 — DB 에 생성하지 않기 때문)
```

---

---

# ⑥ dbt_project.yml 에서 설정

```yaml
models:
  my_project:
    +materialized: view          # 전체 기본값

    staging:
      +materialized: ephemeral   # staging 폴더 전체

    marts:
      +materialized: table       # marts 폴더 전체
```

```
+ 는 하위 폴더에 상속
개별 파일의 config 블록이 우선 적용됨
```

---

---

# ⑦ 선택 가이드

```
staging 모델:
  view        → 빠른 개발 / 간단한 변환
  ephemeral   → DB 에 뷰 만들기 싫을 때 / 중간 로직만

intermediate 모델:
  view / ephemeral

mart 모델:
  table       → BI 도구가 직접 조회하는 테이블
  incremental → 수억 건 이상 대용량

언제 incremental 쓰나:
  매일 쌓이는 로그 / 이벤트 데이터
  전체 재생성에 10분 이상 걸릴 때
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|ephemeral 모델 직접 조회|DB 에 없음|ref() 로 참조하는 모델 통해서만 가능|
|ephemeral 인데 로그에 안 나옴|정상 동작|CTE 로 삽입됨 → 하위 모델 로그 확인|
|incremental 인데 데이터 이상|첫 실행 조건 다름|`--full-refresh` 로 전체 재생성|
|table 인데 변경사항 반영 안 됨|전체 재생성 필요|`dbt run --select 모델명`|