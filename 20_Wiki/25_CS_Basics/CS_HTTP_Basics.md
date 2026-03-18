---
aliases:
  - HTTP
  - Request
  - Response
  - Header
  - Body
  - 무상태성
tags:
  - CS
related:
  - "[[00_CS_HomePage]]"
  - "[[HTTP_Status_Codes]]"
  - "[[Python_Requests_Methods]]"
  - "[[Linux_Network_Basic]]"
  - "[[CS_Operating_System]]"
---

# HTTP_Basics — HTTP 요청과 응답

## 한 줄 요약

```
클라이언트와 서버가 데이터를 주고받는 규칙(프로토콜)
"어떻게 요청하고 어떻게 응답할지" 에 대한 약속
```

---

---

# ① HTTP 란?

```
HyperText Transfer Protocol
웹에서 데이터를 주고받는 표준 통신 규약

클라이언트 (브라우저 / Python requests / 앱)
    ↓  요청 (Request)
서버 (API 서버 / 웹서버)
    ↓  응답 (Response)
클라이언트
```

```
실제 사용 예:
  브라우저에서 URL 입력         → HTTP Request 전송
  공공데이터 API 호출           → HTTP Request 전송
  Kafka Producer → REST API    → HTTP Request / Response
```

---

---

# ② Request — 요청 구조

```
HTTP Request 4가지 구성:
  시작줄 (Start Line)
  헤더   (Headers)
  빈 줄  (Empty Line)
  본문   (Body)
```

```http
GET /api/hospitals?STAGE1=서울 HTTP/1.1
Host: apis.data.go.kr
Authorization: Bearer TOKEN
Content-Type: application/json

{ "key": "value" }
```

## 시작줄

```
메서드   경로 + 쿼리스트링             HTTP 버전
  GET    /api/hospitals?STAGE1=서울    HTTP/1.1
```

## HTTP 메서드

```
GET     데이터 조회   Body 없음   멱등 ⭐️
POST    데이터 생성   Body 있음   비멱등
PUT     전체 수정     Body 있음   멱등
PATCH   일부 수정     Body 있음   비멱등
DELETE  삭제          Body 없음   멱등
```

```
멱등(Idempotent):
  같은 요청을 여러 번 해도 결과가 같음

  GET /users/1  → 100번 호출해도 같은 데이터 반환 ✅
  POST /users   → 100번 호출하면 100개 생성됨     ❌
```

---

---

# ③ Response — 응답 구조

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 128

{ "hpid": "A0000028", "hvec": 5 }
```

```
응답 4가지 구성:
  시작줄: HTTP 버전 + 상태코드 + 상태메시지
  헤더:   응답 메타데이터
  빈 줄
  본문:   실제 데이터 (JSON / XML / HTML)
```

---

---

# ④ Header — 메타데이터

```
본문 외의 부가 정보
Key: Value 형태
```

## 자주 쓰는 요청 헤더

```
Authorization: Bearer TOKEN      인증 토큰
Content-Type: application/json   요청 본문 형식
Accept: application/json         원하는 응답 형식
User-Agent: MyApp/1.0            클라이언트 식별
Host: apis.data.go.kr            요청 대상 서버
```

## 자주 쓰는 응답 헤더

```
Content-Type: application/json   응답 본문 형식
Content-Length: 256              본문 크기 (bytes)
Cache-Control: no-cache          캐시 정책
Set-Cookie: session=abc123       쿠키 설정
```

```python
# requests 에서 헤더 설정
headers = {
    "Authorization": "Bearer TOKEN",
    "Content-Type":  "application/json"
}
r = requests.get(url, headers=headers)

# 응답 헤더 확인
print(r.headers["Content-Type"])   # application/json
```

---

---

# ⑤ Body — 본문

```
실제 전송할 데이터
요청: POST / PUT / PATCH 에서 주로 사용
응답: 서버가 돌려주는 실제 데이터
```

```python
# GET → Body 없음, params 로 전달
r = requests.get(url, params={"STAGE1": "서울"})
# URL: /api?STAGE1=%EC%84%9C%EC%9A%B8  (자동 인코딩)

# POST → Body 에 데이터 포함
r = requests.post(url, json={"name": "Alice"})
# Body: {"name": "Alice"}
```

```
Content-Type 과 Body 형식:
  application/json                   → {"key": "value"}
  application/x-www-form-urlencoded  → key=value&key2=value2
  multipart/form-data                → 파일 업로드
  text/xml                           → <key>value</key>
```

---

---

# ⑥ Status Code — 상태 코드

```
응답 시작줄에 포함된 3자리 숫자
서버가 요청을 어떻게 처리했는지 알려줌
```

```
2xx  성공
  200  OK              정상
  201  Created         생성됨 (POST 성공)
  204  No Content      성공, 본문 없음

3xx  리다이렉션
  301  Moved Permanently  영구 이동
  302  Found              임시 이동

4xx  클라이언트 에러 (내가 잘못)
  400  Bad Request        잘못된 요청
  401  Unauthorized       인증 필요
  403  Forbidden          권한 없음 (인증은 됐지만 접근 불가)
  404  Not Found          리소스 없음
  429  Too Many Requests  요청 너무 많음 (트래픽 초과)

5xx  서버 에러 (서버가 잘못)
  500  Internal Server Error  서버 내부 오류
  503  Service Unavailable    서버 일시 불가
```

```
⚠️ 공공데이터 API 함정:
  HTTP 200 OK 로 응답하지만
  XML Body 안에 에러 코드가 있는 경우
  → resultCode 확인 필수
```

```python
r = requests.get(url, params=params)
r.raise_for_status()   # 4xx / 5xx 면 예외 발생

# 공공데이터 추가 확인
root = ET.fromstring(r.content)
result_code = root.findtext(".//resultCode")
if result_code != "00":
    raise ValueError(f"API 에러: {result_code}")
```

> 전체 상태코드 → [[HTTP_Status_Codes]] 참고

---

---

# ⑦ 무상태성 (Stateless) ⭐️

```
HTTP 의 가장 중요한 특성

각 요청은 독립적
서버는 이전 요청을 기억하지 않음
```

```
유상태 (Stateful):
  클라이언트: "서울 지역 선택했어"
  서버:       "기억할게"
  클라이언트: "병원 목록 줘"
  서버:       "서울 지역 병원 목록이요" ← 이전 상태 기억

무상태 (Stateless) ← HTTP:
  클라이언트: "서울 지역 병원 목록 줘"   ← 매번 전부 포함
  서버:       "서울 지역 병원 목록이요"
  클라이언트: "서울 지역 병원 목록 또 줘" ← 다시 전부 포함
  서버:       "이전 요청 모름, 서울 지역 병원 목록이요"
```

## 장단점

```
장점:
  서버 확장 쉬움 (어떤 서버가 받아도 처리 가능)
  서버 장애 시 다른 서버로 전환 용이
  각 요청이 독립적 → 단순하고 예측 가능

단점:
  매 요청마다 인증 정보를 다시 보내야 함
  요청 크기가 커질 수 있음

해결책:
  쿠키 / 세션 / JWT 토큰으로 상태 유지
```

## DE 실무에서 무상태성 ⭐️

```
Airflow DAG:
  각 Task 는 독립적으로 실행
  이전 Task 상태에 의존하면 안 됨
  → XCom 이나 DB 로 상태 공유

Kafka:
  각 메시지는 독립적
  이전 메시지 내용을 모름
  → 필요한 정보는 메시지에 전부 포함해서 발행

Spark:
  각 파티션은 독립적으로 처리
  다른 파티션 상태를 모름
```

---

---

# ⑧ HTTP vs HTTPS

```
HTTP   평문 전송 → 중간에서 내용 볼 수 있음 ⚠️
HTTPS  TLS/SSL 암호화 → 안전한 전송 ✅

공공데이터 API:
  http://apis.data.go.kr  ← HTTP
  실습에서는 OK
  실제 서비스에서는 HTTPS 사용 권장
```

---

---

# 한눈에 정리

```
Request:
  시작줄  GET /path?query HTTP/1.1
  헤더    Authorization: Bearer TOKEN
  빈 줄
  본문    {"key": "value"}  (POST/PUT 만)

Response:
  시작줄  HTTP/1.1 200 OK
  헤더    Content-Type: application/json
  빈 줄
  본문    {"hpid": "A0000028", "hvec": 5}

무상태성:
  각 요청은 독립적
  서버는 이전 요청을 기억 안 함
  → 매 요청마다 필요한 정보 전부 포함

멱등성:
  GET / DELETE / PUT  → 여러 번 호출해도 결과 같음
  POST                → 호출할 때마다 새로 생성
```