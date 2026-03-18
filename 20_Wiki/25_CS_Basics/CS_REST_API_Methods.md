---
aliases:
  - REST API
  - GET POST PUT DELETE
  - 멱등성
  - CRUD
tags:
  - CS
related:
  - "[[00_CS_HomePage]]"
  - "[[CS_HTTP_Basics]]"
  - "[[Python_Requests_Methods]]"
  - "[[HTTP_Status_Codes]]"
  - "[[CS_TCP_IP]]"
---

# CS_REST_API_Methods — REST API 메서드

## 한 줄 요약

```
HTTP 를 어떻게 쓸지에 대한 설계 원칙
URL = 자원(Resource), Method = 행위(Action)
```

---

---

# ① REST 란?

```
Representational State Transfer
HTTP 프로토콜을 활용해 자원을 주고받는 설계 방식

핵심 원칙:
  URL 은 자원을 표현        /hospitals / /hospitals/123
  Method 는 행위를 표현     GET(조회) / POST(생성) / DELETE(삭제)
  무상태(Stateless)        각 요청은 독립적, 서버가 상태 저장 안 함
```

---

---

# ② HTTP 메서드 — CRUD

|메서드|역할|CRUD|예시|
|---|---|---|---|
|GET|조회|Read|`GET /hospitals`|
|POST|생성|Create|`POST /hospitals`|
|PUT|전체 수정|Update|`PUT /hospitals/123`|
|PATCH|일부 수정|Update|`PATCH /hospitals/123`|
|DELETE|삭제|Delete|`DELETE /hospitals/123`|

```python
import requests

# GET — 데이터 조회
r = requests.get("https://api.example.com/hospitals")

# POST — 데이터 생성
r = requests.post("https://api.example.com/hospitals",
                  json={"name": "서울대병원", "region": "서울"})

# PUT — 전체 수정 (모든 필드 전송)
r = requests.put("https://api.example.com/hospitals/123",
                 json={"name": "서울대병원", "region": "서울", "tel": "02-1234"})

# PATCH — 일부 수정 (바꿀 필드만 전송)
r = requests.patch("https://api.example.com/hospitals/123",
                   json={"tel": "02-5678"})

# DELETE — 삭제
r = requests.delete("https://api.example.com/hospitals/123")
```

---

---

# ③ 멱등성 (Idempotent) ⭐️

```
같은 요청을 여러 번 보내도 결과가 동일한 것

멱등 O:
  GET     몇 번 조회해도 데이터 안 바뀜
  PUT     같은 데이터로 여러 번 수정해도 결과 동일
  DELETE  이미 삭제된 것 다시 삭제해도 "없음" 상태 동일

멱등 X:
  POST    호출할 때마다 새 데이터 생성됨
  PATCH   경우에 따라 다름 (누적 연산이면 멱등 X)
```

```
왜 중요한가:
  네트워크 오류로 요청이 중복 전송될 때
  멱등이면 → 재시도해도 안전
  멱등 아니면 → 중복 생성/처리 위험

  Kafka Producer / Airflow 재시도 설계 시 고려
```

>[[Idempotent(멱등성)]] 내용 참고 

---

---

# ④ PUT vs PATCH

```
PUT   전체 교체 — 보내지 않은 필드는 null/기본값으로 초기화됨
PATCH 일부 수정 — 보낸 필드만 변경, 나머지 유지

예시 (hvec 만 바꾸고 싶을 때):

PUT /hospitals/123
{ "hpid": "A001", "hvec": 5, "hvoc": 3, "region": "서울" }
→ 모든 필드 다 보내야 함 (안 보내면 null 됨)

PATCH /hospitals/123
{ "hvec": 5 }
→ hvec 만 바뀌고 나머지 유지 ✅
```

---

---

# ⑤ REST URL 설계 원칙

```
✅ 좋은 URL:
  GET    /hospitals           전체 병원 목록
  GET    /hospitals/123       특정 병원 조회
  POST   /hospitals           병원 생성
  PATCH  /hospitals/123       병원 수정
  DELETE /hospitals/123       병원 삭제

  GET    /hospitals/123/beds  특정 병원의 병상 정보

❌ 나쁜 URL:
  GET    /getHospitals        메서드를 URL 에 넣지 말 것
  POST   /hospitals/delete    Delete → HTTP DELETE 메서드로
  GET    /hospital_list       소문자 복수형 권장
```

---

---

# ⑥ 공공데이터 API 와의 차이

```
공공데이터 API (국립중앙의료원 등):
  REST 설계 원칙을 엄격히 따르지 않는 경우 많음
  전부 GET 방식으로 조회만 제공
  URL 에 메서드명 포함 (getEmrrmRltmUsefulSckbdInfoInqire)
  → 레거시 방식 (SOAP 스타일)

현대 REST API:
  URL = 자원, Method = 행위로 분리
  JSON 응답
```

---

---

# 한눈에 정리

|메서드|멱등|안전|용도|
|---|---|---|---|
|GET|✅|✅|조회 (서버 상태 변경 없음)|
|POST|❌|❌|생성|
|PUT|✅|❌|전체 수정|
|PATCH|△|❌|일부 수정|
|DELETE|✅|❌|삭제|

```
안전(Safe): 서버 데이터를 변경하지 않는 것
  GET 만 안전 → 조회만 하니까

멱등(Idempotent): 같은 요청 N번 = 1번과 동일한 결과
  GET / PUT / DELETE 는 멱등
  POST 는 멱등 아님
```

---
---
# 엔드포인트 (Endpoint)

```
클라이언트가 요청을 보낼 수 있는 구체적인 URL 주소
"이 주소로 오면 이 기능을 제공한다"

Base URL  + Path        = Endpoint (최종 목적지)
기본 주소   세부 경로
```

```
예시 — 공공데이터 API:

Base URL:
  http://apis.data.go.kr/B552657/ErmctInfoInqireService

Endpoint (Base URL + Path):
  /getEmrrmRltmUsefulSckbdInfoInqire  ← 응급실 가용병상
  /getSrsillDissAceptncPosblInfoInqire ← 중증질환 수용가능
  /getEmrrmSrsillDissMsgInqire         ← 응급실 메시지
  /getEgytBassInfoInqire               ← 병원 기본정보

각 엔드포인트 = 각각 다른 기능
같은 서버(Base URL) 에서 Path 만 달라짐
```


```python
# 코드에서 Base URL 분리해서 관리하는 이유
BASE_URL = "http://apis.data.go.kr/B552657/ErmctInfoInqireService"

# 엔드포인트만 바꿔서 재사용
url1 = f"{BASE_URL}/getEmrrmRltmUsefulSckbdInfoInqire"
url2 = f"{BASE_URL}/getSrsillDissAceptncPosblInfoInqire"
url3 = f"{BASE_URL}/getEmrrmSrsillDissMsgInqire"

# Base URL 이 바뀌면 한 곳만 수정하면 됨
```

```
REST API 엔드포인트 설계 (현대적):

GET    /hospitals          전체 병원 목록 조회
GET    /hospitals/123      특정 병원 조회
POST   /hospitals          병원 생성
PATCH  /hospitals/123      병원 수정
DELETE /hospitals/123      병원 삭제
GET    /hospitals/123/beds 특정 병원 병상 정보

URL = 자원(명사) / Method = 행위(동사)
→ URL 에 동사 넣지 않는 것이 원칙
```