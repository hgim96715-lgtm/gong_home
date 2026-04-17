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
인터파크 내부 API 4개를 순서대로 호출해서
전시 데이터를 수집하고 PostgreSQL 에 적재하는 파이프라인
```

---

---

# ① 전체 흐름

```
main.py --mode crawl
    │
    ▼
crawler_api.py
    ├── STEP 1. 목록 API       → 전시 목록 (goodsCode 리스트)
    ├── STEP 2. Summary API   → 상세 정보 + placeCode 추출
    ├── STEP 3. Place API     → 정확한 주소 / 위경도
    └── STEP 4. 가격 API      → prices_raw (JSONB 원문) + 정규화 리스트
    │
    ▼
load_to_postgres.py
    ├── upsert_exhibitions     → raw_exhibitions
    ├── upsert_exhibition_prices → raw_exhibition_prices
    ├── insert_history         → raw_exhibition_history (일별 스냅샷)
    └── mark_inactive          → is_active = FALSE (이번에 없는 전시)
```

---

---

# ② API 엔드포인트

|API|URL|주요 필드|
|---|---|---|
|목록|`tickets.interpark.com/contents/api/goods/genre`|goodsCode, goodsName, weekRank, startDate|
|Summary|`api-ticketfront.interpark.com/v1/goods/{goodsCode}/summary`|placeCode, playStartDate, dayRank, displayTemplate|
|Place|`api-ticketfront.interpark.com/v1/Place/{placeCode}`|placeAddress, latitude, longitude|
|가격|`api-ticketfront.interpark.com/v1/goods/{goodsCode}/prices/group`|seatGradeName, priceGradeName, salesPrice|

---

---

# ③ Exhibition 데이터클래스

```python
@dataclass
class Exhibition:
    exhibition_id: str       # goodsCode (PK)
    title:         str       # 원본 제목 (접두어 포함)
    subtitle:      str       # subGoodsName
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
    day_rank:      int       # dayRank (INTEGER)
    week_rank:     int       # weekRank (INTEGER)
    month_rank:    int       # monthRank (INTEGER)
    rank:          str       # "주간 1위 / 일간 1위" 합산 텍스트
    image_url:     str       # goodsLargeImageUrl
    detail_url:    str       # 인터파크 상세 페이지
    notice:        str       # displayTemplate 파싱 공지사항
    is_active:     bool      # 크롤링 수집 여부
    crawled_at:    str       # isoformat 타임스탬프
```

---

---

# ④ 핵심 메서드

## _normalize_region — location 정규화

```python
# 목록 API regionName 이 "광주시", "서울특별시" 등으로 옴
# → 행정구역 접미사 제거 → LOCATION_MAP 으로 시/도 매핑

"광주시"      → "광주"
"서울특별시"  → "서울"
"경기도"      → "경기"
```

## _format_date vs _format_datetime

```python
_format_date("20260424")         # → "2026-04-24"  (목록/Summary API 날짜, 8자리)
_format_datetime("202604242359") # → "2026-04-24 23:59"  (가격 API 날짜, 12자리)
# 빈 문자열 / 자릿수 불일치 → None (TIMESTAMP 컬럼 에러 방지)
```

## get_exhibition_detail — 3개 API 통합

```python
def get_exhibition_detail(self, info: dict):
    # 1. Exhibition 객체 목록 API 값으로 초기화
    # 2. Summary API → 덮어씌우기 + place_code 추출
    # 3. Place API   → address / latitude / longitude 저장
    #                  address 로 location 재추출
    # 4. 가격 API    → prices_raw (str) + price_rows (list[dict])
    return ex, price_rows
```

## get_price — 반환값 2개

```python
prices_raw_str, price_rows = crawler.get_price(goods_code)

# prices_raw_str: JSON 문자열 → raw_exhibitions.prices_raw (JSONB)
# price_rows:     list[dict]  → raw_exhibition_prices 정규화 테이블 적재용
```

---

---

# ⑤ load_to_postgres 핵심 패턴

## _to_jsonb — JSONB 컬럼 삽입

```python
# prices_raw 가 str 일 때 → Json() 래핑 필요
# plain str 그대로 넣으면 psycopg2 가 TEXT 로 인식 → 타입 불일치
self._to_jsonb(ex.get("prices_raw"))
# str  → Json(json.loads(value))
# dict/list → Json(value)
# None → None
```

## upsert_exhibition_prices — ON CONFLICT 조건

```sql
ON CONFLICT (exhibition_id, seat_grade, price_grade, price_type_code)
-- init.sql 에 UNIQUE 제약 필수
-- 없으면 ON CONFLICT 실행 시 에러
```

## mark_inactive — active_ids

```python
# 이번 크롤링에 수집된 id 목록
active_ids = [ex["exhibition_id"] for ex in ex_dicts if ex.get("exhibition_id")]

# DB 에서 이 목록에 없는 전시 → is_active = FALSE
loader.mark_inactive(active_ids)
# → != ALL(ARRAY[...]) 사용 (IN 1개짜리 문법 오류 방지)
```

---

---

# ⑥ 실행 명령어

```bash
# 테스트 (DB 연결 + 1건 상세)
python main.py --mode test

# 전체 크롤링
python main.py --mode crawl --pages 10

# API 단독 테스트
python main.py --mode summary-test --goods-code 26002594
python main.py --mode price-test   --goods-code 26002594
python main.py --mode place-test   --place-code 19001136
```

---

---

# ⑦ 자주 나온 에러와 원인

|에러|원인|해결|
|---|---|---|
|`invalid input syntax for type integer`|rank 텍스트가 day_rank INTEGER 컬럼에 들어감|`summary.get("day_rank")` 로 수정|
|`invalid input syntax for type timestamp: ""`|빈 문자열이 TIMESTAMP 컬럼에 들어감|`_format_datetime` 으로 None 변환|
|`INSERT has more expressions than target columns`|DB 테이블에 latitude/longitude 컬럼 없음|`ALTER TABLE` 로 컬럼 추가|
|`ON CONFLICT ... no unique constraint`|UNIQUE 제약 없는 컬럼으로 ON CONFLICT|init.sql 에 UNIQUE 추가|
|`Expecting value: line 1 column 1`|Place API 응답이 JSON 아님 (빈 응답 or HTML)|`resp.text` 로 원문 확인|
|`return` 이 루프 안에 있어 첫 seat_grade 만 반환|get_price 버그|return 을 루프 밖으로 이동|
