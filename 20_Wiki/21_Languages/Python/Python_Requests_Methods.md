---
aliases:
  - GET vs POST
  - HTTP 메서드
  - requests.get
  - requests.post
tags:
  - Python
related:
  - "[[Python_Requests_Response]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[HTTP_Status_Codes]]"
  - "[[00_Python_HomePage]]"
---

## 개념 한 줄 요약

**GET**은 서버에서 데이터를 **"보여줘!"(Read)** 하고 가져오는 방식이고, **POST**는 서버한테 데이터를 **"처리해줘!"(Write/Create)** 하고 보내는 방식입니다. (둘 다 `Response` 객체를 반환합니다.)

---
## 왜 필요한가 (Why)

왜 그냥 하나로 안 쓰고 둘을 나눌까요?

* **GET (엽서):** - 주소창(URL)에 내용이 다 보입니다. 
    - 데이터를 조회할 때 쓰는데, 내용이 공개되어도 상관없는 가벼운 요청에 씁니다. 브라우저 주소창은 **항상 GET** 요청만 보냅니다.
* **POST (편지봉투):** - 봉투 안에 내용을 숨겨서 보냅니다.
    - 로그인(비밀번호)이나 긴 글쓰기처럼 보안이 필요하거나 서버의 상태를 바꿀 때 씁니다.

---
## Practical Context (실무 활용)

현업에서는 이 두 가지 상황을 확실히 구분해야 에러가 안 납니다.

* **GET을 쓸 때:** 네이버 뉴스 검색, 날씨 데이터 가져오기, 깃허브 내 리포지토리 파일 목록 읽어오기
    * **특징:** 백 번을 실행해도 서버 데이터가 변하지 않습니다 (Idempotent 멱등성).> [[Idempotent(멱등성)]] 참조 
* **POST를 쓸 때:** 아이디/비번으로 로그인하기, 쇼핑몰 결제, 슬랙 봇으로 메시지 쏘기
    * **특징:** 실행할 때마다 서버에 새로운 데이터가 생기거나 변합니다.

---
## Code Core Points: 파라미터 4대장 ️

파이썬 `requests` 코드에서 가장 중요한 것은 **데이터를 어디에 실어 보내느냐**입니다.

1. **`params` (GET 전용):** URL 뒤에 `?key=value` 꼬리표처럼 붙여서 보낼 데이터 (딕셔너리)
2. **`data` / `json` (POST 전용):** 남들 안 보이게 요청 몸통(Body)에 숨겨서 보낼 데이터 (딕셔너리)
3. **`headers` (공통 필수):** 신분증이나 편지 봉투의 겉면 정보. (나는 누구고, 어떤 형식의 데이터를 원해!)
4. **`timeout` (공통 필수):** 서버가 응답하지 않을 때 "최대 몇 초까지 기다리다 포기할 것인가?"

---
## Detailed Analysis: 실전 코드 해부

###  GET & POST의 기본 형태 비교

```python
import requests

# --- 1. GET 방식 (조회) ---
url_get = "https://httpbin.org/get"
payload = {'key1': 'value1', 'key2': 'value2'}

# GET은 'params' 인자를 사용해!
# 실제 요청 URL: [https://httpbin.org/get?key1=value1&key2=value2](https://httpbin.org/get?key1=value1&key2=value2)
r = requests.get(url_get, params=payload)


# --- 2. POST 방식 (전송) ---
url_post = "[https://httpbin.org/post](https://httpbin.org/post)"
payload_post = {'username': 'my_id', 'password': 'secret_pw'}

# POST는 'data' (Form 형식) 혹은 'json' (JSON 형식) 인자를 사용해!
r = requests.post(url_post, data=payload_post)
```

### 실전 심화: `headers`와 `timeout`의 비밀 (GitHub API 예시)

외부 API를 호출할 때는 무작정 URL만 찌르면 "너 누구야? 권한 없어!" (401 Unauthorized) 에러를 뱉거나, 무한 대기에 빠질 수 있습니다.

- **`Authorization`:** 비밀번호 대신 사용하는 API 전용 출입증(토큰)입니다.
- **`Accept`:** "나는 결과를 JSON 형식으로 받고 싶어" 라고 서버에 미리 알려주는 옵션입니다.
- **`timeout`:** 서버가 터져서 대답이 없을 때, 무한정 기다리지 않고 "딱 10초만 기다렸다가 에러를 내뿜고 종료해라!"라고 설정하는 필수 안전장치입니다.

```python
import requests
import streamlit as st

url = "[https://api.github.com/repos/hgim96715-lgtm/gong_home/contents/](https://api.github.com/repos/hgim96715-lgtm/gong_home/contents/)"

# 1. Headers 세팅 (신분증 & 메타데이터 챙기기)
headers = {
    "Authorization": f"token {st.secrets['GITHUB_TOKEN']}",
    "Accept": "application/vnd.github.v3+json" 
}

# 2. 실전 API 호출! (신분증 제시하고, 10초 룰 적용)
try:
    r = requests.get(url, headers=headers, timeout=10)
    
    if r.status_code == 200:
        print("성공적으로 데이터를 가져왔습니다!")
    else:
        print(f"에러 발생: {r.status_code}")
        
# 10초가 지나도 답이 없으면 이쪽으로 빠짐 (프로그램 다운 방지)
except requests.exceptions.Timeout:
    print("서버 응답이 너무 늦습니다. 다시 시도해주세요.")
```

---
## 초보자가 자주 착각하는 포인트 (Common Misconceptions)

**"GET으로 긴 데이터 못 보내나?"**
- 네, 못 보냅니다. 브라우저 주소창 길이 제한(약 2,000자)이 있어서 긴 글이나 파일 업로드는 무조건 POST여야 합니다.

**"비밀번호나 토큰을 `params`로 보내면 안 되나요?"**
- **절대 안 됩니다!** `params`는 URL에 그대로 노출되기 때문에, 누군가 링크를 복사하거나 서버 접속 로그(Log)를 보면 토큰이 그대로 유출됩니다.
- 비밀번호나 인증 토큰은 반드시 `headers`의 `Authorization`이나 POST의 `data` 안에 숨겨서 보내야 합니다.

**"`timeout` 안 적어도 알아서 꺼지지 않나요?"**
- 가장 위험한 착각입니다
- `requests` 모듈은 `timeout`을 명시하지 않으면, 서버가 대답할 때까지 **"영원히(무한대)"** 기다립니다.
- 이 때문에 데이터 파이프라인 전체가 마비되는 대참사가 실무에서 종종 발생합니다.
- `timeout=10` (10초) 설정은 선택이 아니라 **필수 생명줄**입니다!