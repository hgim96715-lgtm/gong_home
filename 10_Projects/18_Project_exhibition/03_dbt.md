---
aliases:
  - dbt
tags:
  - Project
related:
  - "[[00_exhibition_Project]]"
  - "[[DBT_Install_Setup]]"
  - "[[DBT_Install_Setup]]"
  - "[[SQL_Data_Types]]"
---
# 03 DBT — Exhibition Project

## 프로젝트 개요

```
인터파크 전시 크롤링 → PostgreSQL (raw) → dbt (transform) → Superset
```

dbt 는 raw 테이블을 **staging → marts** 순서로 변환한다.  
크롤러와 loader 는 데이터를 그대로 적재(EL), dbt 는 변환(T)만 담당한다.

---

## Source 테이블 (raw)

|테이블|설명|주요 컬럼|
|---|---|---|
|`raw_exhibitions`|Summary + Place API 원본|exhibition_id, title, hours, notice, prices_raw (JSONB)|
|`raw_exhibition_prices`|prices/group + bestprices 병합|sales_price, origin_price, discount_rate|
|`raw_exhibition_stats`|연령/성별 통계|age10_rate ~ age50_rate, male_rate, female_rate|
|`raw_exhibition_history`|일별 순위 스냅샷|day_rank, week_rank, month_rank|

---

## Staging 모델

### stg_exhibitions

raw_exhibitions 정제 + 파생 컬럼. CTE 4단계로 구성

```sql
source           → 원본
html_stripped    → HTML 태그 + 엔티티(&nbsp; 등) 제거
hours_extracted  → 관람시간: 키워드 이전 내용 제거
hours_bullet_cleaned → 불릿 대시(-, ·) 제거
staged           → ※ 이후 제거 + 특수기호 제거 + 파생 컬럼
```

**파생 컬럼 정리**

|컬럼|계산|
|---|---|
|`title_cleaned`|`^\s*[\[【...*[\]】...]` 접두어 정규식 제거|
|`duration_days`|`end_date - start_date`|
|`is_currently_on`|`is_active=TRUE AND start_date <= today AND end_date >= today`|
|`is_ended`|`end_date < today`|
|`days_remaining`|`end_date - today` (종료 전시는 NULL)|
|`price_min/max`|`jsonb_array_elements(prices_raw)` 에서 0원 제외 min/max|
|`price_option_count`|가격 항목 수 (0원 제외)|
|`has_discount`|prices_raw 에 '할인' type 포함 여부|

**hours 정제 순서**

```
1. HTML 제거
2. 관람시간: 이전 내용 제거 (전시기간: ... 관람시간: → 관람시간: 이후만)
3. -, · 불릿 제거 (시간 범위 11시 - 19시 는 유지)
4. ※ 이후 제거 (주의사항 제거)
5. 특수기호 + fullwidth 괄호 제거
```

---

### stg_exhibition_stats

`get_top_category` macro 로 `top_age_group` / `gender_dominant` 파생.  
`age10_rate` NULL 8건 → `COALESCE(_, 0)` 처리.

```sql
{{ get_top_category({
    '10대': 'age10_rate',
    '20대': 'age20_rate',
    ...
}) }} AS top_age_group
```

---

### stg_exhibition_prices

`origin_price_calc` 보정 로직:

```sql
CASE
    WHEN price_type_name NOT ILIKE '%할인%'  -- 기본가
        THEN sales_price                      -- sales_price 가 곧 정상가
    WHEN origin_price > 0                    -- bestprices 매핑 성공
        THEN origin_price
    WHEN discount_rate > 0                   -- 역산
        THEN round(sales_price / (1 - discount_rate / 100.0))
    ELSE NULL                                -- 판단 불가
END AS origin_price_calc
```

`sales_price = 0` 인 무료 항목은 `WHERE sales_price > 0` 으로 제외.

---

## Marts (dim 모델)

|모델|설명|주요 용도|
|---|---|---|
|`dim_exhibition_summary`|전체 KPI 1행|대시보드 수치카드|
|`dim_location_analysis`|지역별 분포|지도/막대 차트|
|`dim_price_analysis`|가격대별 분포|가격 분포 차트|
|`dim_current_exhibitions`|진행 중 전시 목록 + stats 조인|전시 목록 테이블|
|`dim_age_gender_analysis`|연령/성별 분포|파이 차트, 연령 차트|


---

## 주요 명령어

```bash
# 컨테이너 접속
docker compose exec dbt bash

# 전체 실행
dbt run

# 모델별 실행
dbt run --select stg_exhibitions
dbt run --select dim_current_exhibitions+  # + = 하위 모델 포함

# 테스트
dbt test

# docs 서버 (포트 8585)
dbt docs generate && dbt docs serve --port 8585

# 컴파일 결과 확인 (SQL 실행 안 함)
dbt compile --select stg_exhibitions
# → target/compiled/ 폴더에 순수 SQL 생성
```

---

## Macro

`macros/get_top_category.sql`

```sql
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

`column_map` = `{'레이블': '컬럼명'}` dict.  
VALUES 인라인 테이블로 만들어 ORDER BY + LIMIT 1 으로 최댓값 레이블 반환.

---

## PENDING

- `dbt run` 실제 실행 및 dim 모델 결과 검증
- Superset 데이터셋 연결
- Airflow DAG 작성 (크롤링 → dbt run 자동화)
- `raw_exhibition_history` 기반 순위 변동 추이 dim 추가