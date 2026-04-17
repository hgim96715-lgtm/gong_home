# 🎨 인터파크 전시 트렌드 분석 데이터 마트

> **인터파크 API 크롤링 → PostgreSQL(Raw) → dbt(Transform) → 데이터 마트 → Superset** ELT 패러다임 기반 전시회 트렌드 분석 파이프라인

---

## 기획 배경

전시회를 자주 보러 다니면서 인터파크에서 전시 정보를 찾아보곤 했는데, 문득 이런 게 궁금해졌습니다.

> "요즘 사람들은 어떤 전시를 많이 볼까?" "지역마다 전시 분포가 다를까?" "가격대별로 어떤 전시가 인기 있을까?" "계절마다 전시 트렌드가 바뀔까?"

인터파크 전시 데이터를 직접 수집하고 분석해서 전시 시장의 트렌드를 데이터로 확인해보고 싶었습니다.

기존 프로젝트(열차, 병원)에서는 **ETL** 방식으로 Spark에서 변환했다면, 이번에는 **ELT** 방식으로 Raw를 먼저 적재하고 dbt로 Transform하는 현대적인 데이터 엔지니어링 패러다임을 직접 경험해보고자 합니다.

---

## ETL vs ELT

|항목|ETL (기존 프로젝트)|ELT (이번 프로젝트)|
|---|---|---|
|Transform 위치|Spark (외부 엔진)|dbt (DB 내부 SQL)|
|Raw 데이터|변환 후 적재|**먼저 적재 후 변환**|
|장점|대용량 분산 처리|유연한 변환, 버전 관리|
|도구|Kafka + Spark|API 크롤링 + dbt|
|적합한 상황|실시간 스트리밍|배치 분석, 데이터 마트|

---

## 기술 스택

|역할|기술|선택 이유|
|---|---|---|
|데이터 수집|Python + requests|인터파크 내부 API 직접 호출 (순수 API)|
|Raw 저장소|PostgreSQL 16|dbt 호환, 기존 경험|
|Transform|dbt Core|SQL 기반, 버전 관리, 문서화 자동화|
|시각화|Apache Superset 3.1|기존 경험|
|스케줄링|Apache Airflow 2.9|dbt 실행 자동화|
|컨테이너|Docker Compose|재현 가능한 환경|

> Selenium / EasyOCR / BeautifulSoup 사용 시도했지만, 
> 인터파크는 스크롤이 아니라 swipe라서 크롤링이 다 안되는 이슈로 인하여 API로 전환 하였습니다. 

---

## 포트

|서비스|포트|
|---|---|
|PostgreSQL|**5435**|
|Airflow|**8085**|
|Superset|**8090**|
|dbt docs|**8585**|

---

## 수집 전략 — 3단계 API

처음에는 Selenium 스크롤로 수집했으나 15건 한계 → DevTools로 내부 API 발굴 → 전국 500건+

```
STEP 1. 목록 API     → 전시 목록 (최대 500개+, 전국)
STEP 2. Summary API  → 상세 정보 (주소/시간/카테고리/공지/이미지/순위)
STEP 3. 가격 API     → prices_raw (JSON 원문, dbt에서 파싱 예정)
```

### API 엔드포인트

|API|URL|주요 필드|
|---|---|---|
|목록|`tickets.interpark.com/contents/api/goods/genre`|goodsCode, goodsName, startDate, endDate, weekRank|
|Summary|`api-ticketfront.interpark.com/v1/goods/{id}/summary`|playTime, displayTemplate, goodsLargeImageUrl|
|가격|`api-ticketfront.interpark.com/v1/goods/{id}/bestprices/group`|priceGradeName, salesPrice|

### 목록 API 파라미터

```python
params = {
    "genre":    "EXHIBIT",
    "page":     page,        # 1~N
    "pageSize": 50,
    "sort":     "WEEKLY_RANKING",
    # region, subCategory 불필요 확인됨
}
```

---

## 수집 데이터 스키마

| 필드              | 출처                               | 설명                                   |
| --------------- | -------------------------------- | ------------------------------------ |
| `exhibition_id` | 목록 API `goodsCode`               | PK                                   |
| `title`         | Summary API `goodsName`          | 원본 타이틀 (접두어 포함)                      |
| `venue`         | Summary API `placeName`          | 전시 장소명                               |
| `location`      | `address` 파싱 → LOCATION_MAP      | 지역 (시/도 단위)                          |
| `address`       | `displayTemplate` 파싱             | 상세 주소                                |
| `period`        | `displayTemplate` 파싱             | 기간 원문 (ex. `26.04.30(목) ~ 26.09.27`) |
| `start_date`    | Summary API `playStartDate`      | 시작일 (DATE)                           |
| `end_date`      | Summary API `playEndDate`        | 종료일 (DATE)                           |
| `hours`         | Summary API `playTime`           | 관람 시간                                |
| `prices_raw`    | 가격 API (JSON)                    | 가격 원문 → dbt에서 파싱                     |
| `age_limit`     | Summary API `viewRateName`       | 관람 연령                                |
| `category`      | Summary API `genreSubName`       | 미술전시 / 전시 / 체험 등                     |
| `rank`          | Summary API 주간/일간/월간 3종          | ex. `주간 3위 / 일간 6위`                  |
| `image_url`     | Summary API `goodsLargeImageUrl` | 고화질 포스터                              |
| `detail_url`    | 목록 API 조합                        | 인터파크 상세 페이지                          |
| `notice`        | `displayTemplate` 파싱             | 얼리버드/사용 안내 등                         |

> `detail_text` 컬럼은 DB에 존재하나 수집 안 함 (OCR 제거, 항상 NULL) `price_adult` / `price_youth` 는 `prices_raw` 로 통합됨

### prices_raw 구조 예시

```json
[
  {"name": "성인", "price": 23000, "type": null, "seat_grade": "입장권"},
  {"name": "어린이/청소년", "price": 18000, "type": null, "seat_grade": "입장권"}
]
```

```json
[
  {"name": "봄맞이 20% 로랑생_성인(유효기간:~4/30)", "price": 18400, "type": "기본가할인", "seat_grade": "입장권"},
  {"name": "봄맞이 20% 로랑생_청소년(유효기간:~4/30)", "price": 14400, "type": "기본가할인", "seat_grade": "입장권"}
]
```

---

## 파일 구조

```
exhibition-data-mart/
├── crawl/
│   ├── crawler_selenium.py   # InterparkCrawler (순수 API)
│   ├── main.py               # 진입점
│   ├── requirements.txt      # requests만 필요
│   └── load/
│       ├── __init__.py
│       └── load_to_postgres.py
├── postgres/
│   └── init.sql              # 테이블 DDL
├── dbt_exhibition/
	├── schema.yml
│   └── models/
│       ├── staging/
│       │   ├── sources.yml
│       │   └── stg_exhibitions.sql
│       └── marts/
│           ├── mart_exhibition_summary.sql
│           ├── mart_location_analysis.sql
│           ├── mart_price_analysis.sql
│           └── mart_current_exhibitions.sql
├── airflow/dags/             # TODO
└── docker-compose.yml
```

---

## 실행 방법

```bash
# 크롤러 테스트
cd crawl
python main.py --mode test

# 가격 API 단독 확인
python main.py --mode price-test --goods-code 25018028

# Summary API 단독 확인
python main.py --mode summary-test --goods-code 26002897

# 전체 크롤링
python main.py --mode crawl --pages 10

# dbt 실행
docker exec -it exhibition-dbt bash
dbt run
dbt test
```

---

## dbt 레이어 구조

```
raw.raw_exhibitions
        │
        ▼ (sources.yml)
raw_staging.stg_exhibitions     ← 정제 + 파생 컬럼
  - title_cleaned               (접두어 제거)
  - price_type                  (prices_raw JSON 파싱)
  - discount_pct                (할인율 추출)
  - rank_number                 (순위 숫자)
  - is_currently_on / is_ended
  - duration_days
        │
        ▼ (ref('stg_exhibitions'))
raw_marts.mart_*
  - mart_exhibition_summary     (KPI)
  - mart_location_analysis      (지역별)
  - mart_price_analysis         (가격대별)
  - mart_current_exhibitions    (현재 진행 중)
        │
        ▼
Superset 대시보드
```

---


---
