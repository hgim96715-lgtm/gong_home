---
aliases:
  - dbt
tags:
  - Project
related:
  - "[[00_exhibition_Project]]"
  - "[[Python_Virtual_Env]]"
  - "[[Python_Database_Connect]]"
  - "[[Python_Type_Checking]]"
  - "[[Python_JSON]]"
  - "[[Python_Classes_Objects]]"
  - "[[SQL_DML_CRUD]]"
---
# 02_crawler_api & load_to_postgres

## 한 줄 요약

```
인터파크 내부 API 5개를 순서대로 호출해서
전시 데이터를 수집하고 PostgreSQL 에 적재하는 파이프라인
```

---

## ① 전체 흐름

```
main.py --mode crawl
    │
    ▼
crawler_api.py
    ├── STEP 1. 목록 API          → 전시 목록 (goodsCode 리스트)
    ├── STEP 2. Summary API      → 상세 정보 + placeCode 추출
    ├── STEP 3. Place API        → 정확한 주소 / 위경도
    ├── STEP 4. 가격 API          → prices_raw (JSONB 원문) + 정규화 리스트
    │             └── bestprices 병합 → originPrice / discountRate 보완
    └── STEP 5. 통계 API          → 연령/성별 비율
    │
    ▼
load_to_postgres.py
    ├── upsert_exhibitions           → raw_exhibitions
    ├── upsert_exhibition_prices     → raw_exhibition_prices
    ├── upsert_stats                 → raw_exhibition_stats
    ├── insert_history               → raw_exhibition_history (일별 스냅샷)
    └── mark_inactive                → is_active = FALSE (이번에 없는 전시)
```

---

## ② API 엔드포인트

|API|URL|주요 필드|
|---|---|---|
|목록|`tickets.interpark.com/contents/api/goods/genre`|goodsCode, goodsName, weekRank, startDate|
|Summary|`api-ticketfront.interpark.com/v1/goods/{goodsCode}/summary`|placeCode, playStartDate, dayRank, displayTemplate|
|Place|`api-ticketfront.interpark.com/v1/Place/{placeCode}`|placeAddress, latitude, longitude|
|가격|`api-ticketfront.interpark.com/v1/goods/{goodsCode}/prices/group`|seatGradeName, priceGradeName, salesPrice|
|bestprices|`api-ticketfront.interpark.com/v1/goods/{goodsCode}/bestprices/group`|originPrice, discountRate|
|통계|`api-ticketfront.interpark.com/v1/statistics/booking`|age10Rate ~ age50Rate, maleRate, femaleRate|

---

## ③ Exhibition 데이터클래스

```python
@dataclass
class Exhibition:
    exhibition_id: str       # goodsCode (PK)
    title:         str       # 원본 제목 (접두어 포함)
    venue:         str       # placeName
    location:      str       # 시/도 단위 (LOCATION_MAP 으로 정규화)
    address:       str       # Place API placeAddress
    latitude:      float     # Place API
    longitude:     float     # Place API
    start_date:    str       # playStartDate YYYY-MM-DD
    end_date:      str       # playEndDate   YYYY-MM-DD
    hours:         str       # playTime (멀티라인)
    prices_raw:    str       # 가격 API 원문 JSON 문자열 → DB 에서 JSONB
    age_limit:     str       # viewRateName
    category:      str       # genreSubName
    genre:         str       # genreName
    day_rank:      int       # dayRank
    week_rank:     int       # weekRank
    month_rank:    int       # monthRank
    rank:          str       # "일간 1위 / 주간 1위 / 월간 1위" 합산 텍스트
    image_url:     str       # goodsLargeImageUrl
    detail_url:    str       # 인터파크 상세 페이지
    notice:        str       # displayTemplate 파싱 공지사항
    is_active:     bool      # 크롤링 수집 여부
    crawled_at:    str       # isoformat 타임스탬프
```

---

## ④ 핵심 메서드

### _extract_location — location 정규화

```python
# Place API address 에서 시/도 추출
# LOCATION_MAP 키워드 매핑
"서울특별시 종로구 ..." → "서울"
"경기도 연천군 ..."     → "경기"
"충청남도 태안군 ..."   → "충남"
```

### _format_date vs _format_datetime

```python
_format_date("20260424")         # → "2026-04-24"      (8자리)
_format_datetime("202604242359") # → "2026-04-24 23:59" (12자리)
# 빈 문자열 / 자릿수 불일치 → None (TIMESTAMP 컬럼 에러 방지)
```

### get_price — bestprices 병합

```python
# prices/group: 전체 grade 목록, originPrice=0
# bestprices/group: originPrice/discountRate 있음, 대표 1건만 반환

# 병합 전략
best_map = {priceGrade: {origin_price, discount_rate}}
fallback_discount = bestprices 첫 항목 discountRate

for row in price_rows:
    best = best_map.get(price_grade, {})
    origin_price  = best.get("origin_price") or 0
    discount_rate = best.get("discount_rate") or 0
    if origin_price == 0 and discount_rate > 0:
        origin_price = round(sales_price / (1 - discount_rate / 100))
```

### get_stats — 통계 API

```python
# 실제 응답 키: age10Rate, age20Rate, maleRate, femaleRate (Rate 붙음)
params = {"goodsCode": goods_code, "placeCode": place_code, "types": "ALL"}
```

---

## ⑤ load_to_postgres 핵심 패턴

### _to_jsonb — JSONB 컬럼 삽입

```python
# prices_raw 가 str 일 때 Json() 래핑 필요
# plain str 그대로 넣으면 psycopg2 가 TEXT 로 인식 → 타입 불일치
self._to_jsonb(ex.get("prices_raw"))
# str  → Json(json.loads(value))
# dict → Json(value)
# None → None
```

### upsert_exhibition_prices — dedup 핵심

```python
# 에러: ON CONFLICT DO UPDATE command cannot affect row a second time
# 원인 1. execute_values 한 번에 완전 중복 행이 들어올 때
# 원인 2. price_type_code 가 None(bestprices) vs ""(prices/group) 로 혼재
#         → Python dict 에서 다른 key → dedup 통과 → DB UNIQUE 충돌

# 수정: dedup 블록 추가 + key 에 or "" 로 None/빈문자열 통일
deduped = {}
for p in prices:
    exhibition_id = p.get("exhibition_id")
    if not exhibition_id:
        continue
    key = (
        exhibition_id,
        p.get("seat_grade")      or "",
        p.get("price_grade")     or "",
        p.get("price_type_code") or "",   # None → "" 통일 ← 핵심
    )
    deduped[key] = p
prices = list(deduped.values())
```

### mark_inactive

```python
# 이번 크롤링에 수집된 id 목록에 없는 전시 → is_active = FALSE
# != ALL(ARRAY[...]) 사용 (NOT IN 문법보다 안전)
UPDATE raw_exhibitions
SET is_active=FALSE, updated_at=NOW()
WHERE exhibition_id != ALL(%s)
AND is_active=TRUE
```

---

## ⑥ 실행 명령어

```bash
# 테스트 (DB 연결 + 1건 상세)
python main.py --mode test

# 전체 크롤링
python main.py --mode crawl --pages 10

# API 단독 테스트
python main.py --mode summary-test --goods-code 26002594
python main.py --mode price-test   --goods-code 26002594
python main.py --mode place-test   --place-code 19001136
python main.py --mode stats-test   --goods-code 26002594
```

---

## ⑦ 자주 나온 에러와 원인

|에러|원인|해결|
|---|---|---|
|`invalid input syntax for type integer`|rank 텍스트가 INTEGER 컬럼에 들어감|`summary.get("day_rank")` 분리|
|`invalid input syntax for type timestamp: ""`|빈 문자열이 TIMESTAMP 컬럼에 들어감|`_format_datetime` → None 변환|
|`INSERT has more expressions than target columns`|DB 에 컬럼 없음|`ALTER TABLE` 로 추가|
|`ON CONFLICT ... no unique constraint`|UNIQUE 제약 없음|init.sql 에 UNIQUE 추가|
|`ON CONFLICT DO UPDATE command cannot affect row a second time`|동일 UNIQUE key 가 한 번에 두 번 들어감 (완전 중복 or None vs "" 혼재)|dedup 블록 추가 + key 에 `or ""` 로 None/빈문자열 통일|
|`'str' object has no attribute 'get'`|통계 API data 가 문자열로 올 때|`if isinstance(d, str): d = json.loads(d)`|
|통계 API 키 오류|`age10` 아닌 `age10Rate`|키 이름 수정|
|Place API 빈 응답|`placeCode`, `types=ALL` 파라미터 누락|params 에 추가|