---
aliases:
  - dbt
  - sources
  - sources.yml
  - source()
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Models]]"
  - "[[DBT_Project_Structure]]"
---

# DBT_Sources — 원본 테이블 관리

## 한 줄 요약

```
sources.yml = "이 테이블은 이미 DB 에 있어. dbt 가 만들진 않지만 참조하고 싶어"
source()    = 모델에서 원본 테이블을 안전하게 참조하는 함수
```

---

---

# ① sources.yml 이 필요한 이유 ⭐️

```
sources.yml 없이:
  원본 테이블을 하드코딩 문자열로 참조
  → 테이블 위치 바뀌면 모든 파일 일일이 수정
  → freshness 체크 불가
  → dbt docs 에서 계보 안 보임

sources.yml 있으면:
  한 곳에서 원본 테이블 정의
  → source('소스명', '테이블명') 으로 참조
  → freshness 모니터링 가능
  → dbt docs 에서 raw → staging → mart 계보 시각화
```

## sources.yml 없을 때 vs 있을 때

```sql
-- ❌ sources.yml 없이 — 하드코딩
SELECT * FROM raw.jaffle_shop.orders

-- ✅ source() 사용 — 중앙 관리
SELECT * FROM {{ source('jaffle_shop', 'orders') }}
```

---

---

# ② sources.yml 작성법 ⭐️

```yaml
# models/staging/sources.yml

version: 2

sources:
  - name: jaffle_shop           # 소스 이름 (source() 첫 번째 인자)
    database: raw               # DB 이름
    schema: jaffle_shop         # 스키마 이름
    tables:
      - name: orders            # 테이블 이름 (source() 두 번째 인자)
      - name: customers

  - name: stripe                # 다른 소스
    database: raw
    schema: stripe
    tables:
      - name: payment
```

```sql
-- 모델에서 사용
SELECT * FROM {{ source('jaffle_shop', 'orders') }}
SELECT * FROM {{ source('jaffle_shop', 'customers') }}
SELECT * FROM {{ source('stripe', 'payment') }}
```

## 계보 흐름

```
sources.yml 정의
  ↓ source('jaffle_shop', 'orders')
stg_orders.sql (staging 모델)
  ↓ ref('stg_orders')
fct_orders.sql (mart 모델)
```

---

---

# ③ freshness 체크 — 데이터 신선도 모니터링

```
freshness = 원본 데이터가 얼마나 최근에 업데이트 됐는지 체크
  warn_after  → 이 시간 지나면 경고
  error_after → 이 시간 지나면 에러
```

```yaml
version: 2

sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    freshness:                    # 소스 전체 기본값
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _etl_loaded_at  # 업데이트 시간 컬럼

    tables:
      - name: orders
        freshness:                # 테이블별 오버라이드
          warn_after: {count: 6, period: hour}
          error_after: {count: 12, period: hour}

      - name: customers           # freshness 없으면 소스 기본값 상속
```

```bash
# freshness 체크 실행
dbt source freshness

# 결과:
# Found source jaffle_shop.orders
#   Loaded at: 2026-04-07 09:00:00 (2 hours ago)
#   Status: PASS ✅

# 12시간 넘으면:
# Status: WARN ⚠️

# 24시간 넘으면:
# Status: ERROR ❌
```

---

---

# ④ 테스트 추가

```yaml
version: 2

sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    tables:
      - name: orders
        columns:
          - name: order_id
            tests:
              - unique
              - not_null
          - name: status
            tests:
              - accepted_values:
                  values: ['placed', 'shipped', 'completed', 'return_pending', 'returned']
```

```bash
dbt test --select source:jaffle_shop   # 소스 테스트만 실행
```

---

---

# ⑤ 컬럼 설명 / 문서화

```yaml
version: 2

sources:
  - name: jaffle_shop
    description: "Jaffle Shop 의 운영 DB"
    database: raw
    schema: jaffle_shop
    tables:
      - name: orders
        description: "고객 주문 데이터"
        columns:
          - name: order_id
            description: "주문 고유 ID"
          - name: customer_id
            description: "고객 ID (customers 테이블 참조)"
          - name: amount
            description: "주문 금액 (센트 단위)"
```

```text
description 작성 기준:
  source 수준  → 이 데이터는 어디서 왔는지 (시스템 설명)
  table 수준   → 이 테이블이 무엇을 담는지
  column 수준  → 이 컬럼이 무엇을 의미하는지 / 단위 명시

dbt docs 에서 자동으로 문서 페이지 생성됨
팀원이 처음 봐도 이해할 수 있도록 작성 권장
```

```bash
dbt docs generate
dbt docs serve
# → sources 도 계보에서 보임
```

---

---

# ⑥ source() 와 ref() 비교 ⭐️

|구분|source()|ref()|
|---|---|---|
|대상|DB 원본 테이블|dbt 모델|
|정의 위치|sources.yml|.sql 파일명|
|사용 위치|주로 staging|staging 이후 모든 모델|
|dbt 가 만드나|❌ 이미 존재|✅ dbt 가 생성|

```sql
-- staging 에서는 source()
SELECT * FROM {{ source('raw', 'orders') }}

-- 이후 모델에서는 ref()
SELECT * FROM {{ ref('stg_orders') }}
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`source()` 에서 테이블 못 찾음|sources.yml 미등록|sources.yml 에 먼저 등록|
|freshness 체크 에러|`loaded_at_field` 없음|업데이트 시간 컬럼 지정|
|staging 에서 `ref()` 사용|source 대신 ref|원본 테이블은 `source()` 사용|
|sources.yml 위치|models/ 밖에 있음|`models/` 폴더 안에 위치|