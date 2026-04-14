---
aliases:
  - api_crawling
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_API_Rate_Limit_Retry]]"
---

---

## 왜 API 크롤링인가

인터파크 전시 프로젝트에서 처음에는 Selenium으로 화면을 스크롤하며 수집했다.

```
Selenium 스크롤 방식
→ Swiper 컴포넌트에 보이는 것만 수집
→ 최대 15~16개 한계
→ 서울 지역만 노출
```

브라우저 DevTools를 열어보니 화면을 렌더링할 때 내부 API를 호출하고 있었다.

```
API 직접 호출 방식
→ pageSize=50, page=1~N 자유 설정
→ 전국 500개+ 수집 가능
→ 응답이 JSON이라 파싱 불필요
```

---

## Step 1 — DevTools로 API 발굴

### 순서

```
1. 크롤링할 페이지를 Chrome에서 열기
2. F12 → Network 탭
3. 상단 필터에서 Fetch/XHR 선택
4. 페이지를 새로고침하거나 버튼 클릭 (다음 페이지, 더보기 등)
5. 목록에 뜨는 요청 중 JSON을 반환하는 것 찾기
6. Headers 탭 → Request URL, Query String Parameters 확인
7. Preview 탭 → 응답 데이터 구조 확인
```

### 인터파크 예시

```
Request URL:
https://tickets.interpark.com/contents/api/goods/genre
    ?genre=EXHIBIT
    &page=2
    &pageSize=25
    &sort=DAILY_RANKING
    &subCategory=08002
    &subCategory=08016
```

> `subCategory`가 배열 파라미터 → requests에서 리스트로 넘기면 자동 처리됨

### 찾은 API가 진짜인지 확인

```python
import requests, json

url = "https://tickets.interpark.com/contents/api/goods/genre"
params = {
    "genre": "EXHIBIT",
    "page": 1,
    "pageSize": 5,
    "sort": "WEEKLY_RANKING",
}
headers = {
    "User-Agent": "Mozilla/5.0 ...",
    "Referer": "https://tickets.interpark.com/contents/genre/exhibition"
}

response = requests.get(url, params=params, headers=headers, timeout=10)
print(response.status_code)                        # 200이면 성공
print(json.dumps(response.json(), indent=2, ensure_ascii=False)[:1000])
```

```text
URL: https://tickets.interpark.com/contents/api/goods/genre?genre=EXHIBIT&page=1&pageSize=10&region=SEOUL&sort=WEEKLY_RANKING
```

```json
요청
:method: GET
:scheme: https
:authority: tickets.interpark.com
:path: /contents/api/goods/genre?genre=EXHIBIT&page=1&pageSize=10&region=SEOUL&sort=WEEKLY_RANKING
Accept: application/json, text/plain, */*
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: ko-KR,ko;q=0.9
Cookie: TodayGoodsList=26005049,Y4001507; _gcl_au=1.1.428742523.1776062210; _kmpid=km|interpark.com|1776062210858|f001a173-f1c1-4f09-908b-341d41b41d8e; tbid=c378208e-a252-4f37-905d-b9e8b14acce3; pcid=177606220980143604
Priority: u=3, i
Referer: https://tickets.interpark.com/contents/genre/exhibition
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/26.5 Safari/605.1.15
```

---

## Step 2 — Session 재사용

`requests.get()`을 매번 호출하면 TCP 연결을 매번 새로 맺는다. `Session`을 쓰면 연결을 유지하고(Keep-Alive), 헤더를 한 번만 설정한다.

```python
import requests

session = requests.Session()
session.headers.update({
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
    "Accept": "application/json",
    "Referer": "https://tickets.interpark.com/contents/genre/exhibition"
})

# 이후 session.get()으로 호출 — 헤더 자동 포함
response = session.get(url, params=params, timeout=10)
```

### Session vs requests.get 비교

|항목|`requests.get()`|`Session`|
|---|---|---|
|TCP 연결|매 요청마다 새로 맺음|재사용 (Keep-Alive)|
|헤더 설정|매번 넘겨야 함|한 번만 설정|
|쿠키 유지|❌|✅ 자동 유지|
|속도|느림|빠름|
|적합한 상황|단발성 1~2회 요청|페이지네이션, 반복 호출|

---

## Step 3 — 페이지네이션 루프

### 기본 패턴

```python
all_items = []
page = 1

while page <= max_pages:
    params = {
        "genre": "EXHIBIT",
        "page": page,
        "pageSize": 50,
        "sort": "WEEKLY_RANKING",
    }

    response = session.get(API_URL, params=params, timeout=10)
    response.raise_for_status()
    data = response.json()

    # 응답 구조가 list인지 dict인지 방어 처리
    if isinstance(data, list):
        items = data
    elif isinstance(data, dict):
        items = data.get("list") or data.get("data", {}).get("list", [])
    else:
        items = []

    if not items:          # 빈 페이지 → 수집 종료
        print("더 이상 항목이 없습니다.")
        break

    all_items.extend(items)
    print(f"페이지 {page}: {len(items)}개 수집")
    page += 1
    time.sleep(0.3)        # 서버 부하 방지 → [[Python_API_Rate_Limit_Retry]] 참고
```

### 종료 조건 3가지

|조건|처리 방법|
|---|---|
|빈 페이지 반환|`if not items: break`|
|최대 페이지 도달|`while page <= max_pages`|
|에러 발생|`except` 블록에서 `break`|

### 응답 구조가 매번 다를 때 방어 처리

```python
# API마다 응답 구조가 다름 → 순서대로 시도
if isinstance(data, list):
    items = data                              # 바로 리스트
elif isinstance(data, dict):
    items = (
        data.get("list")                      # {"list": [...]}
        or data.get("data", {}).get("list", [])  # {"data": {"list": [...]}}
    )
```

> 첫 페이지 응답을 print해서 구조 먼저 확인하는 것이 좋음

---

## Step 4 — 중복 제거

페이지 경계에서 같은 항목이 중복으로 내려오는 경우가 있다.

```python
# 방법 1 — seen set (순서 보장, 가장 일반적)
seen = set()
unique = []
for item in all_items:
    if item["exhibition_id"] not in seen:
        seen.add(item["exhibition_id"])
        unique.append(item)

# 방법 2 — set comprehension (순서 불보장, 간결)
unique = list({item["exhibition_id"]: item for item in all_items}.values())
```

> 인터파크 프로젝트에서는 방법 1 사용 (수집 순서가 랭킹 순서이므로 보존 필요)

---

## Step 5 — dataclass 매핑

API 응답 dict를 바로 쓰면 오타, 누락, 타입 오류가 생기기 쉽다. `dataclass`로 구조를 정의해두면 안전하고 자동완성도 된다.

```python
from dataclasses import dataclass, asdict
from datetime import datetime

@dataclass
class Exhibition:
    exhibition_id: str
    title:         str
    venue:         str  = None
    location:      str  = None
    start_date:    str  = None
    end_date:      str  = None
    image_url:     str  = None
    detail_url:    str  = None
    rank:          str  = None
    age_limit:     str  = None
    crawled_at:    str  = None

    def __post_init__(self):
        # 자동으로 수집 시각 삽입
        if self.crawled_at is None:
            self.crawled_at = datetime.now().isoformat()

    def to_dict(self) -> dict:
        return asdict(self)
```

### API 응답 → dataclass 변환

```python
for item in items:
    goods_code = item.get("goodsCode")
    if not goods_code:
        continue                         # PK 없으면 skip

    # 날짜 포맷 변환: YYYYMMDD → YYYY-MM-DD
    start_date = item.get("startDate", "")
    if start_date and len(start_date) == 8:
        start_date = f"{start_date[:4]}-{start_date[4:6]}-{start_date[6:]}"

    # 이미지 URL 보정: // → https://
    image_url = item.get("posterImageUrl") or item.get("imageUrl", "")
    if image_url.startswith("//"):
        image_url = f"https:{image_url}"

    # 랭킹 포맷 변환
    week_rank = item.get("weekRank")
    rank = f"주간 {week_rank}위" if week_rank else None

    ex = Exhibition(
        exhibition_id = str(goods_code),
        title         = item.get("goodsName", ""),
        venue         = item.get("placeName", ""),
        location      = item.get("regionName"),
        start_date    = start_date,
        image_url     = image_url,
        detail_url    = f"https://tickets.interpark.com/goods/{goods_code}",
        rank          = rank,
        age_limit     = item.get("rateName"),
    )
    results.append(ex)
```

### dict vs dataclass 비교

|항목|dict|dataclass|
|---|---|---|
|오타 감지|❌ 런타임에서야 오류|✅ IDE 자동완성|
|기본값|매번 `.get("key", default)`|선언부에 한 번만|
|직렬화|그대로 사용|`asdict(obj)`|
|`__post_init__`|❌|✅ 초기화 후처리 가능|
|타입 힌트|❌|✅|

---

## 전체 흐름 정리

```
DevTools
  └─ Fetch/XHR에서 API URL + 파라미터 확인
       │
       ▼
Session 생성
  └─ headers (User-Agent, Referer) 한 번만 설정
       │
       ▼
페이지네이션 루프  ←──────────────┐
  ├─ params에 page 번호 증가       │
  ├─ session.get() 호출            │
  ├─ 응답 구조 방어 파싱           │
  ├─ items 없으면 break            │
  └─ time.sleep(0.3) ─────────────┘
       │
       ▼
중복 제거 (seen set)
       │
       ▼
dataclass 매핑
  └─ 날짜 포맷 / URL 보정 / 랭킹 포맷
       │
       ▼
PostgreSQL 적재 → [[Python_Database_Connect]]
```

---

## 자주 쓰는 API 파라미터 패턴

```python
# 배열 파라미터 (requests가 자동으로 ?a=1&a=2 로 변환)
params = {"subCategory": ["08002", "08016"]}

# 날짜 범위
params = {"startDate": "20260101", "endDate": "20261231"}

# 정렬 + 페이지
params = {"sort": "DAILY_RANKING", "page": 1, "pageSize": 50}
```

---