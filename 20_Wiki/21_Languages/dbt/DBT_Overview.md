---
aliases:
  - dbt 개요
  - ETL vs ELT
  - Data Warehouse
  - Data Lake
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Models]]"
---

# DBT_Overview — dbt 란 무엇인가

## 한 줄 요약

```
dbt = SQL 로 데이터를 변환(Transform)하는 도구
      Data Warehouse 안에서 T(Transform) 만 담당
```

---

---

# ① Data Lake vs Data Warehouse

## Data Lake — 원본 저장소

```
원시 데이터(Raw Data)를 원본 형태 그대로 저장
스키마를 읽을 때 적용 (Schema-on-Read)
저렴하고 확장 가능한 스토리지
```

```
저장 데이터 예시:
  JSON 로그 / CSV / Parquet 파일
  이미지 / 동영상
  클릭스트림 이벤트

주요 사용자:
  데이터 엔지니어 / 데이터 사이언티스트 / ML 엔지니어

활용:
  머신러닝 학습 데이터
  탐색적 데이터 분석 (EDA)
  로그 분석
  파이프라인 수집 레이어 (Landing Zone)
```

## Data Warehouse — 분석용 저장소

```
가공 / 정제 / 구조화된 데이터 저장
스키마를 쓸 때 적용 (Schema-on-Write)
빠른 SQL 쿼리에 최적화
```

```
저장 데이터 예시:
  매출(Sales facts)
  고객 정보(Customer dimensions)
  일별 매출 / 월간 활성 유저 집계

주요 사용자:
  데이터 분석가 / BI 도구 / 비즈니스 팀

활용:
  BI 대시보드 / 리포트
  KPI 추적
  Ad-hoc SQL 분석
```

## 비교표

|구분|Data Lake|Data Warehouse|
|---|---|---|
|데이터 형태|원본 그대로 (raw)|정제 / 구조화|
|스키마|읽을 때 (on-read)|쓸 때 (on-write)|
|형식|구조 / 반구조 / 비구조 전부|주로 구조화|
|비용|저렴|상대적으로 비쌈|
|속도|느림|빠름 (SQL 최적화)|
|사용자|엔지니어 / 사이언티스트|분석가 / 비즈니스|

---

---

# ② Data Warehouse 가 필요한 이유

```
운영 DB (Production DB) 는:
  트랜잭션에 최적화 (INSERT / UPDATE / DELETE)
  빠르고 안정적이어야 함
  무거운 분석 쿼리를 위해 설계 안 됨

운영 DB 에서 바로 분석하면:
  애플리케이션 속도 저하
  장애(Outage) 유발
  결과 불일치

→ 분석 전용 저장소(Data Warehouse) 가 필요
```

## OLTP vs OLAP ⭐️

```
OLTP (Online Transaction Processing)
  목적: 트랜잭션 처리 (쓰기 중심)
  예시: 회원가입 / 결제 / 주문 생성
  DB:   운영 DB (PostgreSQL / MySQL 등)

OLAP (Online Analytical Processing)
  목적: 대용량 읽기 / 분석 쿼리
  예시: 국가별 연간 총 매출
  DB:   Data Warehouse (Snowflake / BigQuery / Redshift 등)

Data Warehouse 는 OLAP 을 위해 설계됨
```

## Data Warehouse 계층 구조

```
Raw data         원본 그대로 적재 / 최소한의 변환
    ↓
Cleaned data     타입 수정 / 값 표준화 / NULL 처리
    ↓
Analytics-ready  비즈니스 로직 적용 / 쿼리하기 쉬운 형태

→ dbt 가 이 계층을 SQL 로 관리
```

---

---

# ③ Fact Table vs Dimension Table ⭐️

## Fact Table — 측정 가능한 수치 데이터

```
비즈니스 이벤트를 기록하는 테이블
각 행 = 하나의 비즈니스 이벤트
숫자 / 지표 위주
매우 큰 테이블 (수백만 ~ 수십억 행)
Dimension Table 을 FK 로 참조
```

```
공통 지표 (Metrics):
  revenue        매출
  sales_amount   판매 금액
  quantity       수량
  duration       기간
  count          횟수
```

## Dimension Table — 맥락 설명 데이터

```
Fact 를 설명하는 속성 테이블
텍스트 / 카테고리 위주
상대적으로 작은 테이블
천천히 변함 (Slowly Changing)
필터링 / 그룹핑 / 레이블링에 사용
```

```
공통 Dimension:
  Customer   고객 정보
  Product    상품 정보
  Date       날짜 정보
  Location   위치 정보
  Device     기기 정보
```

## 비교 예시

```sql
-- Fact Table: orders (주문 이벤트)
order_id | customer_id | product_id | date_id | revenue | quantity
  1001   |    C001     |    P100    |  20240101 | 50000  |   2

-- Dimension Table: customers
customer_id | customer_name | city  | signup_date
   C001     |   박이슬      | 서울  | 2023-05-01

-- JOIN 하면:
-- "2024년 1월 1일 서울 거주 고객이 상품 P100 을 2개 구매, 매출 5만원"
```

---

---

# ④ Data Warehouse 데이터 흐름

```
01 Data Origination   소스 시스템에서 데이터 생성
         ↓
02 Data Loading       Data Warehouse 에 적재 (ETL/ELT)
         ↓
03 Data Transformation  분석 가능하도록 가공 ← dbt 가 여기
         ↓
04 Data Analysis      변환된 데이터로 인사이트 도출
```

---

---

# ⑤ ETL vs ELT — dbt 는 어디에?

## ETL (Extract → Transform → Load)

```
소스 → 변환 → DW 적재

변환을 DW 밖에서 먼저 수행
전통적인 방식 (Spark / Python 으로 변환 후 적재)
```

## ELT (Extract → Load → Transform)

```
소스 → DW 적재 → 변환

원본을 먼저 DW 에 적재
변환은 DW 안에서 SQL 로 수행
클라우드 DW 시대의 현대적 방식
```

## dbt 의 역할

```
ELT 에서 T(Transform) 만 담당
DW 안에 이미 있는 데이터를 SQL 로 변환

소스 DB → (Fivetran / Airbyte / Airflow) → DW (raw) → dbt → DW (mart)
                                                              ↑
                                                         여기만 dbt

dbt 가 하는 것:
  SQL SELECT 문으로 변환 모델 정의
  ref() 로 모델 간 의존성 관리
  테스트 자동화
  문서 자동 생성
  데이터 계보(Lineage) 시각화

dbt 가 안 하는 것:
  데이터 수집 (Extraction)
  DW 로 적재 (Loading)
```

---

---

# ⑥ SQL 이 중요한 이유

```
대부분의 Data Warehouse 는 SQL 로 쿼리
SQL 은:
  읽기 쉬움 (Easy to read)
  리뷰하기 쉬움 (Easy to review)
  널리 알려짐 (Widely known)

→ 현대 데이터 파이프라인이 dbt 같은 SQL 기반 도구를 사용하는 이유
```

---

---

# ⑦ Data Warehouse 특징 요약

```
Data is NOT deleted    데이터는 삭제하지 않음
History is preserved   이력을 보존
읽기(Read) 에 최적화
Historical data 저장
Reports / Dashboards / Analytics / ML 에 활용
```