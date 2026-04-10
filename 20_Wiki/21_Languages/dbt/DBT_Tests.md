---
aliases:
  - dbt 테스트
  - accepted_values
  - not_null
  - unique
  - dbt
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Schema_yml]]"
  - "[[DBT_Sources]]"
  - "[[DBT_Custom_Tests]]"
---
# DBT_Tests — 데이터 품질 테스트

## 한 줄 요약

```
dbt test = 데이터가 기대한 대로 들어왔는지 자동 검증
schema.yml 에 선언만 하면 dbt 가 SQL 을 자동 생성해서 실행
```

---

---

# ① 언제 테스트를 추가해야 하나 ⭐️

```
모든 컬럼에 전부 넣을 필요 없음
아래 기준으로 판단

반드시 추가:
  PK 컬럼 (id)          → unique + not_null
  FK 컬럼 (customer_id) → not_null + relationships
  필수 입력값            → not_null

상황에 따라 추가:
  상태 / 유형 컬럼 (status, type, category) → accepted_values
  금액 / 수량 컬럼 (amount, count)          → 커스텀 테스트 (양수 검증)

생략 가능:
  선택적 입력 컬럼 (subtitle, memo 등)
  NULL 이 정상인 컬럼
```

---

---

# ② 내장 테스트 4가지 ⭐️

## not_null — NULL 없음

```yaml
- name: customer_id
  tests:
    - not_null
```

```
이 컬럼에 NULL 이 있으면 FAIL
PK / FK / 필수 입력 컬럼에 사용
```

## unique — 중복 없음

```yaml
- name: customer_id
  tests:
    - unique
```

```
이 컬럼에 중복 값이 있으면 FAIL
PK 컬럼에 반드시 추가
단축 표기: tests: [not_null, unique]
```

## accepted_values — 허용 값 제한 ⭐️

## 구버전 vs 최신 문법 ⭐️

```yaml
# ❌ 구버전 (구형 dbt)
- accepted_values:
    values: ['A', 'B']

# ✅ 최신 dbt 권장 — arguments: 감싸기
- accepted_values:
    arguments:
      values: ['A', 'B']
```

```
dbt 버전 업그레이드 후 구버전 문법 쓰면
경고 또는 에러 발생할 수 있음
→ arguments: 로 감싸는 게 최신 권장 방식
```

## 실전 예시

```yaml
# 불리언
- name: is_active
  tests:
    - accepted_values:
        arguments:
          values: [true, false]

# 문자열 enum
- name: price_type
  tests:
    - accepted_values:
        arguments:
          values: ['normal', 'earlybird', 'earlybird_single', 'single', 'unknown']

# 숫자
- name: rating
  tests:
    - accepted_values:
        arguments:
          values: [1, 2, 3, 4, 5]
```

```
accepted_values 가 필요한 상황:
  컬럼이 정해진 값만 가져야 할 때
  새로운 카테고리가 몰래 들어오면 감지하고 싶을 때
  크롤링 / ETL 오류로 이상한 값이 들어올 수 있을 때

FAIL 조건:
  values 목록에 없는 값이 1건 이라도 있으면 FAIL
  → 어떤 값이 문제인지 출력됨
```

## relationships — FK 관계 검증

```yaml
- name: customer_id
  tests:
    - not_null
    - relationships:
        arguments:
          to: ref('stg_customer')   # 참조 모델
          field: customer_id        # 참조 컬럼
```

```
"이 컬럼의 값이 저 테이블에 반드시 존재해야 함"
→ 데이터 정합성 자동 검증

FAIL 조건:
  customer_id 가 stg_customer 에 없는 값이 있으면 FAIL
```

---

---

# ③ 단축 표기 vs 전체 표기

```yaml
# 단축 표기 — 옵션 없을 때
tests: [not_null, unique]

# 전체 표기 — 옵션 있을 때 (accepted_values / relationships)
tests:
  - not_null
  - unique
  - accepted_values:
      arguments:
        values: ['normal', 'earlybird', 'single']
```

---

---

# ④ accepted_values 실전 패턴 ⭐️

## 불리언

```yaml
- name: is_active
  description: "활성 여부"
  tests:
    - accepted_values:
        arguments:
          values: [true, false]
          # YAML 에서 true/false 는 소문자
          # Python 의 True/False 와 다름 주의
```

## 문자열 enum — 가격 유형

```yaml
- name: price_type
  description: |
    가격 유형:
    - normal: 성인/청소년 분리
    - earlybird: 얼리버드
    - earlybird_single: 얼리버드 단일가
    - single: 단일가
    - unknown: 정보 없음
  tests:
    - accepted_values:
        arguments:
          values: ['normal', 'earlybird', 'earlybird_single', 'single', 'unknown']
```

## 문자열 enum — 주문 상태

```yaml
- name: status
  tests:
    - accepted_values:
        arguments:
          values: ['placed', 'shipped', 'completed', 'returned', 'return_pending']
```

## quote 옵션 — 숫자를 문자로

```yaml
- name: rating_str
  tests:
    - accepted_values:
        values: ['1', '2', '3', '4', '5']
        quote: false    # 값을 따옴표 없이 비교 (숫자형 컬럼)
```

---

---

# ⑤ SQL 로 만든 커스텀 테스트 ⭐️

```
내장 테스트 4가지로 부족할 때
SQL 파일로 직접 테스트 작성
SELECT 결과가 1건 이상이면 FAIL
```

## singular test — 특정 모델 전용

```sql
-- tests/assert_positive_amount.sql
-- amount 가 음수인 건이 있으면 FAIL

SELECT
    payment_id,
    amount
FROM {{ ref('stg_payment') }}
WHERE amount < 0
```

```yaml
# schema.yml 에서 등록 없이 tests/ 에 파일 추가하면 자동 실행
```

## generic test — 재사용 가능

```sql
-- tests/generic/is_positive.sql
-- 어떤 모델, 어떤 컬럼에도 적용 가능

{% test is_positive(model, column_name) %}
SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0
{% endtest %}
```

```yaml
# schema.yml 에서 사용
- name: amount
  tests:
    - is_positive     # 직접 만든 generic test
```

## accepted_values 를 SQL 로 구현하면

```sql
-- dbt 가 내부적으로 이런 SQL 을 자동 생성함
SELECT
    price_type,
    COUNT(*) AS cnt
FROM stg_exhibitions
WHERE price_type NOT IN ('normal', 'earlybird', 'earlybird_single', 'single', 'unknown')
GROUP BY 1
HAVING COUNT(*) > 0
-- 결과가 있으면 FAIL
```

---

---

# ⑥ 테스트 실행

```bash
dbt test                              # 전체 테스트
dbt test --select stg_customer        # 특정 모델만
dbt test --select staging.*           # staging 폴더 전체
dbt test --select source:raw          # sources 테스트만
dbt test --select tag:critical        # 태그 선택
```

## 결과 해석

```
PASS  unique_stg_customer_customer_id        ✅ 중복 없음
PASS  not_null_stg_customer_customer_id      ✅ NULL 없음
PASS  relationships_stg_rental_customer_id   ✅ FK 정합성 OK
FAIL  accepted_values_stg_exhibitions_price_type ❌

FAIL 상세:
  Got 3 values not in accepted list: ['vip', None, '할인']
  ↑ 새로운 값이 들어왔거나 크롤링 오류
```

---

---

# ⑦ 테스트 추가 기준 요약

|컬럼 유형|추천 테스트|
|---|---|
|PK (id)|`unique` + `not_null`|
|FK (customer_id)|`not_null` + `relationships`|
|필수 입력값|`not_null`|
|상태/유형 (status, type)|`accepted_values`|
|선택 입력값|생략 가능|
|금액/수량|커스텀 (is_positive)|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`true/false` 대신 `True/False` 사용|Python 습관|YAML 에서는 `true/false` 소문자|
|`accepted_values` 에 `arguments:` 없음|구버전 문법|최신 dbt 는 `arguments:` 감싸기 필수|
|accepted_values 에 새 값 추가 안 함|데이터 변경 미반영|FAIL 시 values 목록 업데이트|
|PK 에 테스트 없음|귀찮아서 생략|`unique + not_null` 최소한 추가|
|relationships FAIL|상위 모델에 데이터 없음|소스 데이터 / 파이프라인 확인|
|테스트가 너무 많아 느림|모든 컬럼에 추가|핵심 컬럼 위주로 선별|