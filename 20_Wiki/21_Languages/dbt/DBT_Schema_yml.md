---
aliases:
  - schema.yml
  - dbt 테스트
  - dbt 문서화
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Sources]]"
  - "[[DBT_Models]]"
  - "[[DBT_Tests]]"
---
# DBT_Schema_yml — 모델 정의 & 테스트

## 한 줄 요약

```
schema.yml = dbt 모델의 설명 + 테스트를 한 곳에 정의
sources.yml 이 원본 테이블이라면
schema.yml 은 dbt 가 만든 모델을 정의
```

---

---

# ① sources.yml vs schema.yml ⭐️

```
sources.yml:
  DB 에 이미 있는 원본 테이블 정의
  dbt 가 만들지 않은 것
  source() 함수로 참조

schema.yml:
  dbt 가 만든 모델 정의 (stg_ / int_ / dim_ / fct_)
  ref() 함수로 참조
  테스트 + 문서화 담당
```

```
위치:
  models/
  ├── staging/
  │   ├── sources.yml    ← 원본 테이블
  │   └── schema.yml     ← staging 모델 정의
  ├── intermediate/
  │   └── schema.yml     ← intermediate 모델 정의
  └── marts/
      └── schema.yml     ← mart 모델 정의

또는 하나로 통합:
  models/schema.yml      ← 전체 모델 한 파일에
```

---

---

# ② description | — 줄바꿈 있는 긴 설명 ⭐️

```
description: "한 줄 설명"   → 짧은 설명
description: |              → 줄바꿈 포함 여러 줄 설명
  줄1
  줄2

| (파이프) = YAML 블록 스칼라
  줄바꿈을 그대로 유지
  들여쓰기로 범위 결정
```

## 비교

```yaml
# | 없이 — 한 줄만 가능
- name: price_type
  description: "가격 유형: normal / earlybird / single / unknown"

# | 붙이면 — 줄바꿈 + 목록 작성 가능
- name: price_type
  description: |
    가격 유형:
    - normal: 성인/청소년 분리 (신뢰 가능)
    - earlybird: 얼리버드 성인/청소년
    - earlybird_single: 얼리버드 단일가
    - single: 단일가 (구분 없음)
    - unknown: 가격 정보 없음
```

```
| 의 효과:
  dbt docs 에서 줄바꿈이 그대로 렌더링됨
  목록 / 설명 / 주의사항을 깔끔하게 작성 가능
  복잡한 enum 값 설명할 때 특히 유용
```

## 실전 활용 패턴

```yaml
- name: status
  description: |
    주문 상태:
    - placed: 주문 완료
    - shipped: 배송 중
    - completed: 수령 완료
    - returned: 반품

- name: price_type
  description: |
    가격 유형:
    - normal: 성인/청소년 분리 (신뢰 가능)
    - earlybird: 얼리버드 성인/청소년
    - earlybird_single: 얼리버드 단일가
    - single: 단일가 (구분 없음)
    - unknown: 가격 정보 없음

- name: amount
  description: |
    결제 금액 (원 단위).
    음수 불가 / NULL = 무료 입장.
    earlybird 할인 적용 후 금액.
```

---

---

# ③ 기본 구조

```yaml
version: 2

models:
  - name: 모델명          # .sql 파일명과 동일
    description: "설명"
    columns:
      - name: 컬럼명
        description: "컬럼 설명"
        tests:
          - unique
          - not_null
```

---

---

# ④ 테스트 종류 ⭐️

## 단축 표기 vs 전체 표기

```yaml
# 단축 표기 (간단할 때)
- name: rental_id
  tests: [not_null, unique]

# 전체 표기 (옵션 있을 때)
- name: rental_id
  tests:
    - not_null
    - unique
```

## 4가지 내장 테스트

```yaml
tests:
  - unique          # 중복 없음
  - not_null        # NULL 없음
  - accepted_values:
      values: [True, False]   # 허용 값 목록
  - relationships:
      arguments:
        to: ref('stg_customer')   # 참조 모델
        field: customer_id        # 참조 컬럼
```

```
relationships = FK 관계 검증
  "이 컬럼의 값이 저 테이블의 저 컬럼에 반드시 존재해야 함"
  데이터 정합성 자동 검증
```

---

---

# ⑤ 레이어별 통합 schema.yml ⭐️

```yaml
# models/schema.yml
# staging / intermediate / marts 전부 한 파일에

version: 2

models:
  # ─────────────────────────
  # STAGING
  # ─────────────────────────
  - name: stg_customer
    description: "dvdrental.customer 원본 정제"
    columns:
      - name: customer_id
        description: "PK (dvdrental.customer)"
        tests:
          - not_null
          - unique
      - name: store_id
        tests: [not_null]
      - name: email
        description: "고객 이메일 (일부 NULL 가능)"
      - name: is_active
        description: "활성 여부 (True/False)"
        tests:
          - accepted_values:
              values: [True, False]

  - name: stg_rental
    description: "dvdrental.rental 원본 정제"
    columns:
      - name: rental_id
        tests: [not_null, unique]
      - name: customer_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customer')
                field: customer_id

  - name: stg_payment
    description: "dvdrental.payment 원본 정제"
    columns:
      - name: payment_id
        tests: [not_null, unique]
      - name: customer_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customer')
                field: customer_id
      - name: rental_id
        tests:
          - relationships:
              arguments:
                to: ref('stg_rental')
                field: rental_id

  - name: stg_inventory
    description: "dvdrental.inventory 원본 정제"
    columns:
      - name: inventory_id
        tests: [not_null, unique]
      - name: film_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_film')
                field: film_id

  # ─────────────────────────
  # INTERMEDIATE
  # ─────────────────────────
  - name: int_rentals_enriched
    description: "rental + film + payment 통합 집계"
    columns:
      - name: rental_id
        tests: [not_null, unique]
      - name: customer_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customer')
                field: customer_id
      - name: film_id
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_film')
                field: film_id

  # ─────────────────────────
  # MARTS
  # ─────────────────────────
  - name: dim_customer
    description: "고객 Dimension (surrogate key + 정제 속성)"
    columns:
      - name: customer_sk
        tests: [not_null, unique]
      - name: customer_id
        tests: [not_null, unique]

  - name: dim_film
    description: "영화 Dimension (surrogate key + 속성)"
    columns:
      - name: film_sk
        tests: [not_null, unique]
      - name: film_id
        tests: [not_null, unique]

  - name: fct_payments
    description: "{{ doc('fct_payments_doc') }}"
    columns:
      - name: customer_sk
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_customer')
                field: customer_sk
      - name: film_sk
        tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_film')
                field: film_sk
```

---

---

# ⑥ relationships 테스트 — FK 검증 ⭐️

```
staging 에서:
  customer_id → ref('stg_customer') 검증
  rental_id   → ref('stg_rental') 검증

marts 에서:
  customer_sk → ref('dim_customer') 검증
  film_sk     → ref('dim_film') 검증

계층이 내려갈수록 상위 모델을 참조
→ 데이터 정합성이 레이어 전체에서 보장됨
```

```bash
# 테스트 실행
dbt test                              # 전체
dbt test --select stg_customer        # 특정 모델만
dbt test --select staging.*           # staging 전체
dbt test --select tag:mart            # 태그로 선택
```

---

---

# ⑦ {{ doc() }} — 긴 설명 분리

```
description 이 길어지면 yaml 파일이 지저분해짐
→ docs/ 폴더에 별도 .md 파일로 분리
→ {{ doc('문서명') }} 으로 참조
```

```yaml
# schema.yml 에서 참조
- name: fct_payments
  description: "{{ doc('fct_payments_doc') }}"
```

```markdown
<!-- docs/fct_payments.md -->
{% docs fct_payments_doc %}
결제 Fact 테이블.
고객 / 영화 / 렌탈 정보를 통합한 분석용 테이블.
Surrogate key 로 dim 테이블과 조인.
{% enddocs %}
```

---

---

# ⑧ 테스트 실행 & 결과

```bash
dbt test

# 출력 예시:
# PASS unique_stg_customer_customer_id ✅
# PASS not_null_stg_customer_customer_id ✅
# PASS relationships_stg_rental_customer_id ✅
# FAIL accepted_values_stg_customer_is_active ❌
#   Got 1 values not in accepted list: [None]
```

```
PASS → 테스트 통과
FAIL → 데이터 품질 문제 발견
       어떤 값이 문제인지 출력됨
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`name` 이 .sql 파일명과 다름|오타|파일명과 정확히 일치|
|relationships 테스트 실패|FK 데이터 불일치|소스 데이터 품질 확인|
|`ref()` 대신 테이블명 하드코딩|실수|`to: ref('모델명')` 형식|
|테스트 없이 배포|schema.yml 미작성|핵심 컬럼에 최소 not_null 추가|
|`{{ doc() }}` 파일 못 찾음|docs/ 폴더 위치 틀림|models/ 폴더 안에 위치|