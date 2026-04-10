---
aliases:
  - dbt macro
  - Jina
  - 매크로
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Models]]"
  - "[[DBT_Project_Structure]]"
---

# DBT_Macros — 매크로 & Jinja 문법

## 한 줄 요약

```
macro = SQL 에서 쓰는 재사용 가능한 함수
Jinja = macro 를 작성하는 템플릿 언어 (Python 문법과 유사)
{{ }} 안에 넣으면 dbt 가 SQL 실행 전에 처리함
```

---

---

# ① {{ }} 이게 뭔가 ⭐️

```
dbt 의 SQL 파일은 순수 SQL 이 아님
Jinja 템플릿 언어가 섞여 있음
dbt run 할 때 {{ }} 안의 내용을 먼저 처리하고 → SQL 실행

Jinja 블록 3가지:
  {{ }}   표현식 — 값을 출력 / 함수 호출
  {% %}   제어문 — if / for / macro 정의
  {# #}   주석   — SQL 실행 전에 제거됨
```

## {{ }} — 표현식 (값 출력)

```sql
-- 함수 호출 결과를 SQL 에 삽입
SELECT * FROM {{ source('raw', 'orders') }}
-- dbt 가 → SELECT * FROM raw.public.orders 로 변환

SELECT * FROM {{ ref('stg_orders') }}
-- dbt 가 → SELECT * FROM public.stg_orders 로 변환

{{ safe_trim('first_name') }} AS first_name
-- dbt 가 → nullif(trim(first_name), '') AS first_name 로 변환
```

## {% %} — 제어문 (로직)

```sql
-- macro 정의
{% macro say_hello() %}
    'hello from dbt'
{% endmacro %}

-- if 조건
{% if target.type == 'snowflake' %}
    trim({{ expr }})
{% else %}
    ltrim(rtrim({{ expr }}))
{% endif %}

-- incremental 조건
{% if is_incremental() %}
WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

## {# #} — 주석

```sql
{# 이 쿼리는 2026-04-09 에 추가됨 #}
SELECT * FROM {{ ref('stg_orders') }}
-- dbt 컴파일 후 주석 제거됨
```

## 컴파일 과정

```
.sql 파일 (Jinja 포함)
  ↓ dbt compile
순수 SQL ({{ }} 처리 완료)
  ↓ DB 전송
실행 결과
```

```bash
# 컴파일 결과 확인 (실행 안 함)
dbt compile --select my_model
# target/compiled/ 폴더에 순수 SQL 생성됨
```

---

---

# ② macro 란 ⭐️

```
macro = SQL 에서 쓰는 재사용 함수
반복되는 SQL 로직을 함수로 만들어 한 번만 정의
모든 모델에서 {{ macro_name() }} 으로 호출

macros/ 폴더에 .sql 파일로 저장
```

## 가장 간단한 예시

```sql
-- macros/say_hello.sql
{% macro say_hello() %}
    'hello from dbt'
{% endmacro %}

-- 모델에서 사용
SELECT {{ say_hello() }} AS greeting;

-- 컴파일 결과
SELECT 'hello from dbt' AS greeting;
```

---

---

# ③ 실전 macro 예시 — safe_trim ⭐️

```
문제: 텍스트 컬럼에 공백이나 빈 문자열("") 이 자주 섞임
     JOIN 이나 집계 시 "" 와 NULL 이 다르게 처리됨
해결: 공백 제거 + 빈 문자열 → NULL 변환 macro
```

## macro 정의

```sql
-- macros/safe_trim.sql

{% macro safe_trim(expr) -%}
    nullif(trim({{ expr }}), '')
{%- endmacro %}
```

```
인자: expr → 컬럼명 또는 표현식
동작:
  trim({{ expr }})       → 앞뒤 공백 제거
  nullif(..., '')        → 빈 문자열 → NULL 변환

{%- -%} 의 - :
  줄바꿈 / 공백 트리밍 (컴파일 결과를 깔끔하게)
```

## 모델에서 사용

```sql
-- models/staging/stg_customers.sql

SELECT
    customer_id,
    store_id,
    {{ safe_trim('first_name') }}  AS first_name,
    {{ safe_trim('last_name') }}   AS last_name,
    {{ safe_trim('email') }}       AS email,
    activebool                     AS is_active,
    create_date,
    last_update
FROM {{ source('dvdrental', 'customer') }}
```

## 컴파일 결과

```sql
-- dbt run 할 때 실제 실행되는 SQL
SELECT
    customer_id,
    store_id,
    nullif(trim(first_name), '') AS first_name,
    nullif(trim(last_name), '')  AS last_name,
    nullif(trim(email), '')      AS email,
    activebool                   AS is_active,
    create_date,
    last_update
FROM raw.public.customer
```

---

---

# ④ Jinja 문법 정리 ⭐️

## 공백 트리밍 — - 옵션

```
{% macro %} → 앞뒤 공백 유지
{%- macro %} → 앞 공백 제거
{% macro -%} → 뒤 공백 제거
{%- macro -%} → 양쪽 공백 제거
```

## if / else / endif

```sql
{% if 조건 %}
    -- 참일 때
{% elif 조건2 %}
    -- 조건2 참
{% else %}
    -- 거짓
{% endif %}
```

## 크로스 DB 분기 — target.type

```sql
{% macro trim_whitespace(expr) %}
    {% if target.type == 'snowflake' %}
        trim({{ expr }})
    {% else %}
        ltrim(rtrim({{ expr }}))
    {% endif %}
{% endmacro %}
```

```
target.type = 현재 연결된 DB 종류
  'postgres' / 'snowflake' / 'bigquery' / 'redshift'

DB 마다 함수가 다를 때 macro 로 추상화
```

---

---

# ⑤ 왜 macro 를 쓰나

```
Reusability (재사용성):
  같은 로직을 50개 모델에 복사하지 않고 한 번만 작성

Consistency (일관성):
  모든 모델이 같은 방식으로 문자열 정제
  같은 로직 / 같은 edge case / 버그 줄어듦

Cross-database support:
  macro 안에서 DB 별 분기 → 모델 코드는 동일하게 유지
```

---

---

# ⑥ 실무에서 자주 쓰는 macro 패턴

```
surrogate_key     → 여러 컬럼을 합쳐 PK 생성
safe_trim         → 공백 제거 + 빈 문자열 → NULL
date_spine        → 연속된 날짜 시퀀스 생성
audit_columns     → created_at / updated_at 자동 추가
adapter_specific  → DB 별 함수 분기

→ dbt_utils 패키지에 많이 구현돼 있음
   pip install dbt-utils 으로 사용 가능
```

---

---

# ⑦ macro 위치 & 호출

```bash
# 폴더 구조
macros/
├── safe_trim.sql
├── surrogate_key.sql
└── utils/
    └── date_helpers.sql
```

```sql
-- 호출: 파일명이 아닌 macro 이름으로 호출
{{ safe_trim('column_name') }}
{{ surrogate_key(['col1', 'col2']) }}

-- 다른 파일에 있어도 자동으로 찾음
-- import 불필요
```

---

---

# Best Practices ⭐️

```
✅ DO:
  macro 는 작고 하나의 역할에만 집중
  이름을 명확하게 (safe_trim / O, do_stuff / X)
  비슷한 기능끼리 그룹화 (macros/string/, macros/date/)
  dbt_utils 패키지 먼저 확인 (이미 구현된 것 많음)

❌ DON'T:
  비즈니스 로직을 macro 에 넣지 말 것
    → 모델 SQL 에 직접 작성하는 게 가독성 좋음
  macro 가 전체 SELECT 문을 반환하게 하지 말 것
    → 컬럼 수준의 변환만
  가독성이 중요한 SQL 에 남용하지 말 것
    → 남이 읽기 어려워짐
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`{{ macro() }}` 결과가 빈 값|macro 에 return 없음|macro 본문에 변환할 SQL 작성|
|macro 못 찾음|파일이 macros/ 밖에 있음|macros/ 폴더 안에 위치|
|컴파일 결과 공백 이상|`{%- -%}` 없음|공백 트리밍 옵션 추가|
|DB 별로 다른 결과|target.type 분기 없음|`{% if target.type %}` 로 분기|