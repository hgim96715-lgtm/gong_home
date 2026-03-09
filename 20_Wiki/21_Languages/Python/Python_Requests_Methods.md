---
aliases:
  - requests
  - GET POST
  - requests.get
  - requests.post
  - requests.Session
  - Session
  - params
  - headers
  - timeout
  - Authorization
  - RequestException
  - JSONDecodeError
  - 멱등성
  - Idempotent
  - params 자동 변환
tags:
  - Python
related:
  - "[[Python_Requests_Response]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[HTTP_Status_Codes]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Error_Handling]]"
  - "[[Python_URL_Parsing]]"
---
# Python_Requests_Methods

## 개념 한 줄 요약

> **GET : "보여줘!" (Read) — POST : "처리해줘!" (Write/Create)** 둘 다 `Response` 객체를 반환한다.

|구분|GET|POST|
|---|---|---|
|**비유**|엽서 (내용이 보임)|편지봉투 (내용이 숨겨짐)|
|**데이터 위치**|URL 뒤 `?key=value`|요청 Body 안|
|**용도**|조회 (뉴스, 날씨, API)|생성/변경 (로그인, 결제)|
|**멱등성**|O (몇 번 호출해도 서버 불변)|X (호출할 때마다 서버 변경)|

---

---

# ① GET / POST 기본 형태

```python
import requests

# GET: params 로 URL 뒤에 붙임
r = requests.get(
    "https://httpbin.org/get",
    params={"key1": "value1", "key2": "value2"},
    timeout=10
)
# 실제 요청 URL: https://httpbin.org/get?key1=value1&key2=value2

# POST: data (Form) 또는 json (JSON) 으로 Body 에 숨김
r = requests.post(
    "https://httpbin.org/post",
    data={"username": "my_id", "password": "secret_pw"},
    timeout=10
)

# POST: JSON 형식으로 보낼 때
r = requests.post(
    "https://httpbin.org/post",
    json={"name": "Alice", "age": 30},  # Content-Type 자동으로 application/json 설정
    timeout=10
)
```

---

---

# ② 파라미터 4대장

|파라미터|용도|위치|
|---|---|---|
|`params`|GET 쿼리스트링|URL 뒤|
|`data`|POST Form 데이터|Body|
|`json`|POST JSON 데이터|Body (Content-Type 자동 설정)|
|`headers`|신분증 / 메타데이터|요청 헤더|
|`timeout`|최대 대기 시간 (초)|—|

## params 의 숫자 자동 변환 ⭐️

> API 공식 문서에 `string` 이라고 적혀 있어도 `params` 에 숫자(int) 를 그대로 넣어도 완벽하게 작동한다.

```python
# API 문서에 pageNo, numOfRows 가 string 이라고 명시돼 있어도
params = {
    "serviceKey" : API_KEY,
    "pageNo"     : 1,      # int 그대로 넣어도 OK
    "numOfRows"  : 100,    # int 그대로 넣어도 OK
    "dataType"   : "JSON",
    "run_ymd"    : "20260308",
}
```

```
requests 가 서버로 보내기 직전에
params 딕셔너리의 모든 값을 자동으로 문자열(str) 로 변환한다.

1     ->  "1"
100   ->  "100"
True  ->  "True"

실제 전송 URL: ?pageNo=1&numOfRows=100&...
```

```
그래서:
str(pageNo) 로 직접 변환할 필요 없음
"1" 로 바꿀지 1 로 놔둘지 고민할 필요 없음
가독성을 위해 숫자는 숫자 타입 그대로 쓰는 게 더 깔끔
```

---
## params 의 함정 — 특수문자 인코딩 ⭐

>️`params={}` 는 편리하지만 **특수문자가 포함된 파라미터 이름** 에서 문제가 생긴다.

```python
# 이렇게 넘기면
params = {"cond[run_ymd::EQ]": "20260308"}

# requests 가 파라미터 이름까지 인코딩해버림
# 실제 전송: ?cond%5Brun_ymd%3A%3AEQ%5D=20260308
#                ↑ [ → %5B   ] → %5D   : → %3A
# -> 서버가 알아보지 못하고 400 Bad Request 반환
```

```text
원인: requests 는 params 딕셔너리의 key 도 URL 인코딩 대상으로 처리함. 
[] :: 같은 특수문자가 key 에 포함되면 전부 %XX 로 변환됨.
```

```python
# 해결: URL 을 직접 f-string 으로 조립
query_str = f"pageNo=1&numOfRows=100&cond[run_ymd::EQ]=20260308"
url = f"{base_url}/{endpoint}?serviceKey={api_key}&returnType=JSON&{query_str}"

# 실제 전송: ?serviceKey=...&returnType=JSON&pageNo=1&cond[run_ymd::EQ]=20260308
# 대괄호가 그대로 유지됨 -> 서버가 정상 처리
```

```text
언제 params={} 를 쓰고 언제 직접 조립하나?

파라미터 이름이 평범한 영문/숫자  -> params={} 그냥 사용
파라미터 이름에 [] :: 등 특수문자 -> f-string 직접 조립
```

>[[02_API_Producer]] 코레일 V2 API 에서 실제로 겪은 문제. [[Python_URL_Parsing]] 인코딩 참고


---

# ③ API 문서 읽기 — 필수 vs 선택 파라미터

## 별표(*) 가 전부다

```
API 문서에서 파라미터 이름 옆 별표(*) 유무만 확인하면 됨

serviceKey *   <- * 있음 = 필수 (안 넣으면 응답 자체를 안 해줌)
pageNo         <- * 없음 = 선택 (안 넣으면 서버 기본값으로 처리)
numOfRows      <- * 없음 = 선택
run_ymd        <- * 없음 = 선택
dptre_stn_cd   <- * 없음 = 선택
```

```
식당 주문 비유:
필수(*): 메뉴 자체 (없으면 주문 불가)
선택  : 추가 토핑 (안 고르면 기본으로 나옴)
```

## 함수 파라미터에 뭘 넣어야 하나 ⭐️

```
함수 파라미터 = "호출할 때마다 바뀌는 값" 만 넣는다.

넣어야 하는 것  : 호출할 때마다 달라지는 값 (날짜, 역 코드 등)
넣지 않아도 되는 것: 항상 고정인 값 (serviceKey, dataType, pageNo 등)
                    -> params 딕셔너리 안에 하드코딩
```

```python
# ❌ 처음에 헷갈려서 이렇게 짜고 싶었던 것
def get_train_schedule(self, serviceKey, pageNo, numOfRows, dataType, run_ymd, dptre_stn_cd):
    ...
# 호출할 때마다 serviceKey, pageNo, dataType 을 매번 넘겨줘야 해서 불편

# ✅ 올바른 방식
def get_train_schedule(self, run_ymd: str, dptre_stn_cd: str = "0001") -> list:
    params = {
        "serviceKey" : self.api_key,   # 항상 고정 -> 함수 밖에서 관리
        "pageNo"     : 1,              # 항상 고정 -> params 에 하드코딩
        "numOfRows"  : 100,            # 항상 고정 -> params 에 하드코딩
        "dataType"   : "JSON",         # 항상 고정 -> params 에 하드코딩
        "run_ymd"    : run_ymd,        # 호출마다 다름 -> 파라미터로 받음
        "dptre_stn_cd": dptre_stn_cd,  # 기본값 있지만 바꿀 수 있음 -> 기본값 파라미터
    }
```

## 파라미터 분류 기준

|분류|넣는 곳|예시|
|---|---|---|
|항상 고정|`params` 딕셔너리에 하드코딩|`pageNo=1`, `dataType="JSON"`|
|호출마다 다름|함수 파라미터 (필수)|`run_ymd`|
|보통 고정인데 가끔 바꿀 수 있음|함수 파라미터 (기본값 지정)|`dptre_stn_cd="0001"`|
|인증 키|`self.api_key` 또는 `.env`|`serviceKey`|

---

---

# ④ headers — 신분증

```python
headers = {
    "Authorization": f"Bearer {API_TOKEN}",   # API 인증 토큰
    "Accept": "application/json",              # JSON 형식으로 받겠다
    "Content-Type": "application/json",        # 내가 보내는 것도 JSON
}

r = requests.get(url, headers=headers, timeout=10)
```

## Authorization 형식

```text
Bearer 토큰   # OAuth, JWT 등 대부분의 API
token 토큰    # GitHub API
Basic 토큰    # ID:PW 를 Base64 인코딩한 것
```

>`params` 에 토큰을 넣으면 URL 에 그대로 노출된다. 
>서버 로그, 브라우저 기록에 남아서 유출 위험. 인증 정보는 **반드시 headers 의 Authorization** 에 넣어야 한다.

---
---
# ⑤ timeout — 필수 안전장치

```python
# timeout 없으면 서버가 응답할 때까지 영원히 대기
# 파이프라인 전체가 멈추는 대참사 발생

# 연결 10초, 응답 30초 각각 설정
r = requests.get(url, timeout=(10, 30))

# 둘 다 같은 값으로 설정
r = requests.get(url, timeout=10)
```


---

---

# ⑥ requests.Session — 같은 서버에 반복 호출할 때 ⭐️

> 매번 `requests.get()` 을 호출하면 매번 새로운 TCP 연결을 맺는다. `Session` 을 쓰면 연결을 재사용해서 **속도가 빠르고**, headers / auth 를 한 번만 설정하면 된다. API 를 반복 호출하는 Producer 에서 필수.

```python
import requests

# Session 없이: 매 요청마다 연결 새로 맺음
r1 = requests.get(url1, headers=headers, timeout=10)
r2 = requests.get(url2, headers=headers, timeout=10)  # headers 반복

# Session 사용: 연결 재사용 + 공통 설정 한 번만
session = requests.Session()
session.headers.update({
    "Authorization": f"Bearer {API_TOKEN}",
    "Accept": "application/json",
})

# session.get() 안에 들어갈 수 있는 것들
session.get(
    url,                           # 필수: 요청 URL
    params={"key": "value"},       # 선택: 쿼리스트링 파라미터
    headers={"X-Extra": "info"},   # 선택: 추가 헤더 (Session 헤더에 병합됨)
    timeout=10,                    # 선택: 대기 시간
)

# 자주 쓰는 패턴
r1 = session.get(url1)                                   # Session 설정 그대로
r2 = session.get(url2, params={"run_ymd": "20260308"})   # 파라미터만 추가
r3 = session.get(url3, timeout=5)                        # 이 요청만 timeout 다르게

# with 문으로 자동 종료 (권장)
with requests.Session() as session:
    session.headers.update({"Authorization": f"Bearer {API_TOKEN}"})
    r = session.get(url, params={"key": "value"}, timeout=10)
    data = r.json()
```

## Session 활용 — 공공데이터 API 반복 호출

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv("TRAIN_API_KEY")
BASE_URL = "http://apis.data.go.kr/1613000/TrainInfoService"

def fetch_train_data(dates: list) -> list:
    results = []

    with requests.Session() as session:
        # 공통 파라미터 (매 요청마다 반복할 것들)
        base_params = {
            "serviceKey" : API_KEY,
            "pageNo"     : 1,
            "numOfRows"  : 100,
            "dataType"   : "JSON",
            "dptre_stn_cd": "0001",
        }

        for date in dates:
            params = {**base_params, "run_ymd": date}  # 날짜만 바꿔서 요청
            r = session.get(f"{BASE_URL}/getStrtpntAlocFndTrainInfo",
                            params=params, timeout=10)
            r.raise_for_status()
            results.append(r.json())

    return results
```

---

---

# ⑦ 예외 처리 — requests.exceptions ⭐️

```python
from requests.exceptions import (
    RequestException,   # 모든 requests 예외의 부모 클래스
    ConnectionError,    # 네트워크 연결 실패
    Timeout,            # timeout 초과
    HTTPError,          # 4xx, 5xx 응답 (raise_for_status 에서 발생)
    JSONDecodeError,    # 응답이 JSON 이 아닐 때
)
```

## 예외 계층 구조

```
RequestException          <- 최상위 (이것 하나로 전부 잡을 수 있음)
├── ConnectionError       <- 서버 연결 자체 실패 (DNS, 네트워크)
├── Timeout               <- 설정한 timeout 초과
├── HTTPError             <- raise_for_status() 가 던지는 예외
└── JSONDecodeError       <- r.json() 실패 (응답이 HTML/XML 이거나 빈 값)
```

## raise_for_status() — 상태코드 자동 검사 ⭐️

> `requests.get()` 은 4xx, 5xx 에러 응답이 와도 **예외를 발생시키지 않는다.** `r.raise_for_status()` 를 호출해야 비로소 HTTPError 가 발생한다.

```python
r = requests.get(url, params=params, timeout=10)

# raise_for_status() 없이는 에러 응답을 모르고 지나침
print(r.status_code)  # 404 가 와도 그냥 진행됨
print(r.json())       # 에러 응답을 정상 응답처럼 처리하는 대참사

# raise_for_status() 호출하면
r.raise_for_status()
# 200 OK          -> 아무 일도 안 일어남 (통과)
# 400 Bad Request -> HTTPError 발생
# 401 Unauthorized-> HTTPError 발생
# 404 Not Found   -> HTTPError 발생
# 500 Server Error-> HTTPError 발생
```

```
상태코드 범위:
2xx (200~299) -> 통과
4xx (400~499) -> HTTPError (클라이언트 잘못)
5xx (500~599) -> HTTPError (서버 잘못)
```

## 실전 예외 처리 패턴

```python
import requests
from requests.exceptions import RequestException, Timeout, JSONDecodeError

def fetch_api(url: str, params: dict) -> dict | None:
    try:
        r = requests.get(url, params=params, timeout=10)
        r.raise_for_status()  # 4xx, 5xx 이면 HTTPError 발생

        # 간혹 API 오류 시 정상(200) 응답이지만
        # body 가 XML 또는 HTML 로 반환되어 JSON 디코딩 에러 발생
        # -> JSONDecodeError 로 따로 잡아줘야 함
        return r.json()

    except Timeout:
        print(f"[Timeout] {url} 응답 없음 (10초 초과)")
        return None

    except JSONDecodeError:
        # 공공데이터 API 에서 자주 발생
        # API 키 오류 -> XML 에러 메시지 반환
        # 트래픽 초과 -> HTML 반환
        print(f"[JSONDecodeError] JSON 이 아님: {r.text[:200]}")
        return None

    except RequestException as e:
        # ConnectionError, HTTPError 등 나머지 전부 잡음
        print(f"[RequestException] {type(e).__name__}: {e}")
        return None
```

## JSONDecodeError 가 나는 상황

```python
# 공공데이터 API 에서 자주 발생하는 케이스:
# 1. API 키 오류   -> 200 OK 이지만 XML 에러 메시지 반환
# 2. 트래픽 초과   -> HTML 응답 반환
# 3. 서버 점검 중  -> 빈 응답

# XML 반환 예시 (공공데이터 API 키 오류)
# <?xml version="1.0" encoding="UTF-8"?>
# <OpenAPI_ServiceResponse>
#   <cmmMsgHeader>
#     <errMsg>SERVICE KEY IS NOT REGISTERED ERROR.</errMsg>
#   </cmmMsgHeader>
# </OpenAPI_ServiceResponse>

# 디버깅: r.text 로 원본 응답 먼저 확인
r = requests.get(url, params=params, timeout=10)
print(r.status_code)  # 200 인데도 XML 이 올 수 있음
print(r.text[:300])   # 원본 텍스트 확인

try:
    data = r.json()
except JSONDecodeError:
    print("JSON 파싱 실패. 원본 응답:", r.text[:200])
```

---

---

# 초보자 실수 체크리스트

| 실수                         | 원인              | 해결                                 |
| -------------------------- | --------------- | ---------------------------------- |
| 인증 토큰을 `params` 에 넣음       | URL 에 노출됨       | `headers["Authorization"]` 에 넣기    |
| `timeout` 미설정              | 무한 대기           | 항상 `timeout=10` 이상 설정              |
| `r.json()` 에서 에러           | 응답이 JSON 이 아님   | `JSONDecodeError` 처리 + `r.text` 확인 |
| 반복 호출에 매번 `requests.get()` | 매번 새 연결         | `requests.Session()` 사용            |
| `raise_for_status()` 미호출   | 4xx 에러를 모르고 지나침 | 항상 호출 후 처리                         |
