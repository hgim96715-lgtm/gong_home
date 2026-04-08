---
aliases:
  - dbt 프로젝트 구조
  - dbt_project.yml
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Install_Setup]]"
  - "[[DBT_Models]]"
---

# DBT_Project_Structure — 프로젝트 구조

## 한 줄 요약

```
dbt 프로젝트 = SQL 파일들을 체계적으로 관리하는 폴더 구조
dbt_project.yml = 프로젝트 전체 설정 파일 (뇌)
```

---

---

# ① 전체 폴더 구조

```
my_dbt_project/
├── dbt_project.yml      ← 프로젝트 설정 (핵심)
├── profiles.yml         ← DB 연결 설정 (~/.dbt/ 에 위치)
│
├── models/              ← SQL 변환 모델 (핵심)
│   ├── staging/         ← 원본 데이터 가공 (1단계)
│   ├── intermediate/    ← 중간 변환 (2단계)
│   └── marts/           ← 분석용 최종 테이블 (3단계)
│
├── seeds/               ← CSV → DB 테이블
├── tests/               ← 커스텀 테스트 SQL
├── macros/              ← 재사용 Jinja 함수
├── snapshots/           ← SCD Type 2 이력 관리
├── analyses/            ← 분석용 SQL (빌드 안 함)
│
└── target/              ← 컴파일된 SQL (자동 생성 / git 제외)
```

---

---

# ② dbt_project.yml — 프로젝트 설정 파일 ⭐️

```yaml
# 프로젝트 기본 정보
name: 'learn_dbt'      # 프로젝트 이름 (소문자 + 언더스코어)
version: '1.0.0'       # 버전

# profiles.yml 의 어떤 프로필 사용할지
# profiles.yml 의 최상위 키와 반드시 일치해야 함
profile: 'learn_dbt'

# 각 폴더 경로 설정
model-paths:    ["models"]     # SQL 변환 모델
analysis-paths: ["analyses"]   # 분석용 SQL (빌드 안 함)
test-paths:     ["tests"]      # 커스텀 테스트
seed-paths:     ["seeds"]      # CSV 파일
macro-paths:    ["macros"]     # Jinja 매크로
snapshot-paths: ["snapshots"]  # SCD Type 2

# dbt clean 할 때 삭제할 폴더
clean-targets:
  - "target"        # 컴파일된 SQL
  - "dbt_packages"  # 설치된 패키지
```

## models 설정 — Materialization ⭐️

```yaml
models:
  learn_dbt:                    # 프로젝트 이름
    +materialized: view         # 기본값: 전체 모델을 view 로
    staging:
      +materialized: ephemeral  # staging 폴더: ephemeral
    marts:
      +materialized: table      # marts 폴더: table
```

```
Materialization 종류:
  view       → SQL 뷰 생성 (빠름 / 매번 계산)
  table      → 실제 테이블 생성 (느림 / 저장됨)
  ephemeral  → 임시 CTE (DB 에 저장 안 함 / 다른 모델이 참조만)
  incremental → 증분 업데이트 (대용량 테이블)

+ 는 하위 폴더에 상속된다는 의미
```

>ephermral를 성정하면 dbt는 이를 데이터베이스에 독립적인 테이블이나 뷰로 구축(Build)하지 않기 때문에 DataGrip 이나 테이블이나 뷰로 보이지 않음 유의!! 

## vars 설정 — 전역 변수

```yaml
vars:
  lookback_interval: '3 days'   # 전역 변수 정의
```

```sql
-- 모델에서 사용
WHERE created_at >= CURRENT_DATE - INTERVAL '{{ var("lookback_interval") }}'
```

```bash
# 실행 시 오버라이드
dbt run --vars '{"lookback_interval": "7 days"}'
```

## seeds 설정

```yaml
seeds:
  learn_dbt:
    +quote_columns: false    # 컬럼명에 따옴표 안 붙임
```

---

---

# ③ models/ — SQL 변환 모델 ⭐️

```
dbt 의 핵심
SELECT 문만 작성하면 dbt 가 자동으로 테이블/뷰 생성
ref() 함수로 모델 간 의존성 관리
```

## 레이어별 역할

```
models/
├── staging/      원본 데이터를 정제 (rename / cast / clean)
├── intermediate/ 여러 staging 모델 조합 (비즈니스 로직)
└── marts/        분석가가 쓰는 최종 모델 (집계 / 요약)
```

## 모델 파일 예시

```sql
-- models/staging/stg_orders.sql

SELECT
    order_id,
    customer_id,
    order_date::DATE         AS order_date,
    amount::NUMERIC(10, 2)   AS amount,
    status
FROM {{ source('raw', 'orders') }}   -- 원본 테이블 참조
WHERE status != 'cancelled'
```

```sql
-- models/marts/fct_orders.sql

SELECT
    o.order_id,
    o.order_date,
    o.amount,
    c.customer_name,
    c.region
FROM {{ ref('stg_orders') }} o        -- 다른 모델 참조
JOIN {{ ref('stg_customers') }} c
    ON o.customer_id = c.customer_id
```

```
ref()    → 다른 dbt 모델 참조 (의존성 자동 관리)
source() → 원본 DB 테이블 참조
```

## 모델별 설정 오버라이드

```sql
-- 파일 맨 위에 config 블록으로 설정 변경
{{ config(
    materialized='table',
    schema='analytics',
    tags=['daily', 'finance']
) }}

SELECT ...
```

---

---

# ④ seeds/ — CSV → DB 테이블

```
작은 참조 데이터 (매핑 테이블 / 코드표 등)
CSV 파일을 DB 테이블로 자동 적재
코드로 관리 가능 (git 추적)
```

```
seeds/
└── country_codes.csv
└── product_categories.csv
```

```csv
-- seeds/country_codes.csv
country_code,country_name
KR,South Korea
US,United States
JP,Japan
```

```bash
dbt seed                        # 전체 seed 실행
dbt seed --select country_codes # 특정 seed 만
```

```sql
-- 모델에서 seed 참조
SELECT *
FROM {{ ref('country_codes') }}
```

---

---

# ⑤ tests/ — 커스텀 테스트

```
schema.yml 의 내장 테스트(not_null, unique 등) 외에
직접 SQL 로 작성하는 테스트

테스트 실패 조건: SELECT 결과가 1건 이상이면 실패
```

```sql
-- tests/assert_positive_amount.sql
-- 주문 금액이 음수인 경우 → 테스트 실패

SELECT order_id, amount
FROM {{ ref('fct_orders') }}
WHERE amount < 0
```

```bash
dbt test                          # 전체 테스트
dbt test --select fct_orders      # 특정 모델 테스트만
```

---

---

# ⑥ macros/ — 재사용 Jinja 함수

```
반복되는 SQL 로직을 함수로 만들어두고 재사용
Jinja 템플릿 문법 사용
```

```sql
-- macros/cents_to_dollars.sql

{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::NUMERIC(10, 2)
{% endmacro %}
```

```sql
-- 모델에서 사용
SELECT
    order_id,
    {{ cents_to_dollars('amount_cents') }} AS amount_dollars
FROM {{ ref('stg_orders') }}
```

---

---

# ⑦ snapshots/ — 이력 관리 (SCD Type 2)

```
천천히 변하는 데이터의 변경 이력 추적
고객 주소 / 상품 가격 변경 이력 등
변경 전/후 값을 모두 보존
```

```sql
-- snapshots/customer_snapshot.sql

{% snapshot customer_snapshot %}

{{ config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at'
) }}

SELECT * FROM {{ source('raw', 'customers') }}

{% endsnapshot %}
```

```bash
dbt snapshot
```

---

---

# ⑧ analyses/ — 분석용 SQL (빌드 안 함)

```
탐색적 쿼리 / 임시 분석 SQL
dbt run 에서 제외 (빌드 안 함)
dbt compile 로 Jinja 렌더링만 가능
```

```sql
-- analyses/revenue_by_region.sql
SELECT
    region,
    SUM(amount) AS total_revenue
FROM {{ ref('fct_orders') }}
GROUP BY region
```

---

---

# 자주 쓰는 명령어

```bash
dbt run                           # 전체 모델 실행
dbt run --select staging.*        # staging 폴더만
dbt run --select +fct_orders      # fct_orders + 의존 모델
dbt test                          # 전체 테스트
dbt seed                          # CSV 적재
dbt snapshot                      # 스냅샷 실행
dbt compile                       # SQL 컴파일 (실행 안 함)
dbt clean                         # target/ 삭제
dbt docs generate && dbt docs serve  # 문서 생성 + 서버
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`profile` 이름 불일치|`dbt_project.yml` 과 `profiles.yml` 다름|두 파일의 이름 일치시키기|
|모델이 빌드 안 됨|`model-paths` 경로 틀림|`dbt_project.yml` 경로 확인|
|`ref()` 모델 못 찾음|파일명 오타|파일명과 `ref('이름')` 일치 확인|
|seed 컬럼 따옴표 에러|`quote_columns` 설정|`+quote_columns: false`|
|테스트가 항상 통과|SELECT 결과 0건 = 성공|의도적으로 실패 케이스 데이터 넣기|