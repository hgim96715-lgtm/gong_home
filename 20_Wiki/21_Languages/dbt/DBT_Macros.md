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

# ⑧ Jinja for 루프 ⭐️

```
{% for %} 를 쓰면 반복되는 SQL 을 동적으로 생성할 수 있음
Python 의 for 문과 거의 동일한 문법
```

## 기본 문법

```jinja
{% for item in list %}
    {{ item }}
{% endfor %}
```

## loop 내장 변수

```jinja
{% for item in list %}
    {{ loop.index }}    {# 현재 인덱스 (1부터) #}
    {{ loop.index0 }}   {# 현재 인덱스 (0부터) #}
    {{ loop.first }}    {# 첫 번째 항목이면 true #}
    {{ loop.last }}     {# 마지막 항목이면 true #}
    {{ loop.length }}   {# 전체 항목 수 #}
{% endfor %}
```

## 쉼표 처리 패턴 ⭐️

```
SQL 에서 가장 흔한 문제:
마지막 항목에는 쉼표를 붙이면 안 됨

패턴 1: loop.last 로 조건 처리
  {{ "," if not loop.last }}

패턴 2: loop.index0 > 0 으로 앞에 붙이기
  {{ ", " if loop.index0 > 0 }}col_name
```

## 실전 예시 — 컬럼 목록 동적 생성

```sql
{% macro select_columns(columns) %}
    {% for col in columns %}
        {{ col }}{{ ", " if not loop.last }}
    {% endfor %}
{% endmacro %}

-- 사용
SELECT {{ select_columns(['id', 'name', 'price']) }}
FROM raw_exhibitions

-- 컴파일 결과
SELECT id, name, price
FROM raw_exhibitions
```

---

---

# ⑨ dict + for 루프 — VALUES 패턴 ⭐️

```
column_map 같은 dict 를 인자로 받아
VALUES 절을 동적으로 생성하는 패턴
실제 내가 만든 get_top_category macro 의 핵심 구조
```

## dict.items() — key, value 꺼내기

```python
# Python 과 동일
column_map = {
    "10대": "age10_rate",
    "20대": "age20_rate",
    "30대": "age30_rate",
}

for label, col in column_map.items():
    print(label, col)
# "10대" "age10_rate"
# "20대" "age20_rate"
# "30대" "age30_rate"
```

```jinja
{# Jinja 도 동일 문법 #}
{% for label, col in column_map.items() %}
    {{ label }}, {{ col }}
{% endfor %}
```

## VALUES 절 동적 생성

```sql
-- VALUES 절은 여러 행을 직접 나열하는 SQL 문법
SELECT *
FROM (
    VALUES
        ('10대', 1.1),
        ('20대', 13.3),
        ('30대', 36.2)
) AS v(category_name, category_value)
```

```
VALUES 를 Jinja for 루프로 동적 생성하면:
  컬럼이 늘어도 macro 인자만 바꾸면 됨
  코드 중복 없음
```

## 내가 만든 get_top_category macro 전체 분석

```sql
-- macros/get_top_category.sql

{% macro get_top_category(column_map) %}
    (
        SELECT category_name
        FROM (
            VALUES
                {% for label, col in column_map.items() %}
                    ('{{ label }}', COALESCE({{ col }}, 0)){{ "," if not loop.last }}
                {% endfor %}
        ) AS v(category_name, category_value)
        ORDER BY category_value DESC
        LIMIT 1
    )
{% endmacro %}
```

```
구조 분해:

  인자: column_map → dict 로 받음
    key   = 레이블 문자열   ('10대', '20대', ...)
    value = 컬럼명 문자열   ('age10_rate', ...)

  VALUES 절:
    ('{{ label }}', COALESCE({{ col }}, 0))
    →  label → '10대'     (따옴표로 감싸서 문자열 리터럴)
    →  col   → age10_rate (따옴표 없이 → 컬럼명으로 인식)

  COALESCE({{ col }}, 0):
    NULL 이면 0 으로 처리 (rate 가 NULL 인 경우 방지)

  {{ "," if not loop.last }}:
    마지막 행에는 쉼표 없음 (SQL 문법 오류 방지)

  ORDER BY category_value DESC LIMIT 1:
    가장 높은 비율의 레이블 1개 반환
```

## 컴파일 결과 예시

```sql
-- 호출
SELECT
    exhibition_id,
    {{ get_top_category({
        "10대": "age10_rate",
        "20대": "age20_rate",
        "30대": "age30_rate",
        "40대": "age40_rate",
        "50대": "age50_rate"
    }) }} AS top_age_group
FROM stg_exhibition_stats

-- 컴파일 결과
SELECT
    exhibition_id,
    (
        SELECT category_name
        FROM (
            VALUES
                ('10대', COALESCE(age10_rate, 0)),
                ('20대', COALESCE(age20_rate, 0)),
                ('30대', COALESCE(age30_rate, 0)),
                ('40대', COALESCE(age40_rate, 0)),
                ('50대', COALESCE(age50_rate, 0))
        ) AS v(category_name, category_value)
        ORDER BY category_value DESC
        LIMIT 1
    ) AS top_age_group
FROM stg_exhibition_stats
```

## 같은 macro 로 성별 우세도 구할 수 있음

```sql
-- 성별 최다 그룹
{{ get_top_category({
    "남성": "male_rate",
    "여성": "female_rate"
}) }} AS gender_dominant
```

---

## AS v(category_name, category_value) — 인라인 테이블 컬럼 이름 붙이기 ⭐️

### 왜 필요한가

```sql
-- VALUES 로 만든 인라인 테이블은 컬럼 이름이 없음
SELECT *
FROM (
    VALUES
        ('10대', 1.1),
        ('20대', 13.3)
)
-- 에러: column1, column2 같은 자동 이름이 붙거나
--       ORDER BY category_value 같은 참조가 불가능함
```

```sql
-- AS v(컬럼명1, 컬럼명2) 로 이름을 직접 지정
SELECT *
FROM (
    VALUES
        ('10대', 1.1),
        ('20대', 13.3)
) AS v(category_name, category_value)
--   ↑        ↑               ↑
-- 테이블 별칭  첫 번째 컬럼명   두 번째 컬럼명

-- 이제 컬럼명으로 참조 가능
ORDER BY category_value DESC
```

### 문법 구조

```sql
(...서브쿼리 또는 VALUES...) AS 테이블별칭(컬럼명1, 컬럼명2, ...)
--                              ↑
--                          괄호 안에 컬럼 이름 순서대로 나열
--                          VALUES 의 값 순서와 1:1 매핑
```

### VALUES 값 순서 ↔ 컬럼명 매핑

```sql
VALUES
    ('10대', 1.1),   -- ('category_name', category_value)
    ('20대', 13.3)
AS v(category_name, category_value)

-- ('10대', 1.1)
--    ↓       ↓
-- category_name = '10대'
-- category_value = 1.1
```

### get_top_category macro 에서의 역할

```sql
(
    SELECT category_name          -- ← 여기서 category_name 을 참조
    FROM (
        VALUES
            ('10대', COALESCE(age10_rate, 0)),
            ('20대', COALESCE(age20_rate, 0)),
            ('30대', COALESCE(age30_rate, 0))
    ) AS v(category_name, category_value)
    --   ↑  첫 번째 값('10대' 등) → category_name
    --      두 번째 값(비율)      → category_value
    ORDER BY category_value DESC  -- ← 여기서 category_value 를 참조
    LIMIT 1
)
```

```
흐름:
  1. VALUES 로 (레이블, 비율) 쌍을 행으로 나열
  2. AS v(category_name, category_value) 로 컬럼명 부여
  3. ORDER BY category_value DESC → 비율이 높은 순서로 정렬
  4. LIMIT 1 → 가장 높은 비율의 행만 선택
  5. SELECT category_name → 그 행의 레이블 반환
```

### 컬럼 수가 다르면 에러

```sql
-- ❌ VALUES 에 컬럼이 2개인데 이름을 3개 지정 → 에러
VALUES ('10대', 1.1) AS v(name, value, extra)

-- ❌ VALUES 에 컬럼이 3개인데 이름을 2개 지정 → 에러
VALUES ('10대', 1.1, 'test') AS v(name, value)

-- ✅ VALUES 컬럼 수 = 이름 수
VALUES ('10대', 1.1) AS v(category_name, category_value)
```

### 일반 서브쿼리에도 동일하게 적용

```sql
-- SELECT 결과에도 컬럼명 지정 가능
SELECT *
FROM (
    SELECT location, COUNT(*) AS cnt
    FROM raw_exhibitions
    GROUP BY location
) AS region_summary(location, exhibition_count)
--  ↑ 기존 컬럼명(location, cnt) 을 재명명
--    → AS 절 없이 괄호로 새 이름 부여
```

```
일반적으로는 SELECT 절에서 AS 로 이름 붙임
VALUES 처럼 컬럼명이 아예 없을 때 이 문법이 필수
```

### Oracle (SQLD) vs PostgreSQL 차이

| |PostgreSQL|Oracle|
|---|---|---|
|인라인 테이블 별칭|`AS v(col1, col2)`|`v(col1, col2)` (AS 없음)|
|VALUES 지원|✅|Oracle 12c+ ✅|

```sql
-- Oracle
SELECT * FROM (
    SELECT 1 AS num FROM DUAL
    UNION ALL
    SELECT 2 FROM DUAL
) v(number_col)   -- AS 없음

-- PostgreSQL
SELECT * FROM (
    VALUES (1), (2)
) AS v(number_col)  -- AS 필수
```

---

---

# ⑩ VALUES 절 — SQL 기본 개념

```
VALUES 는 테이블 없이 직접 행을 나열하는 문법
주로 인라인 테이블로 활용
```

```sql
-- 기본: 행 직접 나열
VALUES
    ('서울', 30),
    ('부산', 5),
    ('대구', 3)

-- FROM 절에서 인라인 테이블로 사용
SELECT region, cnt
FROM (
    VALUES
        ('서울', 30),
        ('부산', 5)
) AS v(region, cnt)

-- WITH 절과 함께
WITH region_data AS (
    SELECT *
    FROM (
        VALUES ('서울', 30), ('부산', 5)
    ) AS v(region, cnt)
)
SELECT * FROM region_data
```

```
dbt macro 에서 VALUES 를 쓰는 경우:
  동적으로 (레이블, 값) 쌍을 만들어 집계할 때
  CASE WHEN 대신 ORDER BY + LIMIT 1 로 최댓값 레이블 뽑을 때
  여러 컬럼을 언피벗(unpivot) 할 때
```

---

---

# ⑪ macro 를 stg 에 적용하는 방법

## stg_exhibition_stats.sql 개선 예시

```sql
-- ❌ 기존 — CASE WHEN 중첩 반복
case
    when greatest(...) = coalesce(age10_rate,0) then '10대'
    when greatest(...) = coalesce(age20_rate,0) then '20대'
    ...
end as top_age_group

-- ✅ macro 사용 — 한 줄로 해결
{{ get_top_category({
    "10대": "age10_rate",
    "20대": "age20_rate",
    "30대": "age30_rate",
    "40대": "age40_rate",
    "50대": "age50_rate"
}) }} AS top_age_group
```

```
macro 쓰는 기준:
  같은 패턴이 2곳 이상 반복된다 → macro 로 추출
  인자만 다르고 구조가 동일하다 → macro 화 가능

macro 쓰지 말아야 할 때:
  비즈니스 로직 (조건이 복잡하고 예외가 많을 때)
  한 번만 쓰는 경우 (오히려 가독성 저하)
  팀원이 macro 를 모를 때 (유지보수 어려움)
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
