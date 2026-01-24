---
aliases:
  - GET vs POST
  - HTTP 메서드
  - requests.get
  - requests.post
tags:
  - Python
  - HTTP
  - GET
  - POST
related:
  - "[[Python_Requests_Response]]"
  - "[[Pandas_Json_Normalize]]"
---
##  개념 한 줄 요약

**GET**은 서버에서 데이터를 **"보여줘!"(Read)** 하고 가져오는 방식이고,
**POST**는 서버한테 데이터를 **"처리해줘!"(Write/Create)** 하고 보내는 방식이야.
GET, POST 둘다 Respose 객체를 반환한다 

---
## 왜 필요한가 (Why)

왜 그냥 하나로 안 쓰고 둘을 나눌까?

**GET (엽서):** 
- 주소창(URL)에 내용이 다 보여. 
- 데이터를 조회할 때 쓰는데, 내용이 공개되어도 상관없는 가벼운 요청에 써.
- 브라우저 주소창은 **항상 GET** 요청만 보냄.

**POST (편지봉투):** 
- 봉투 안에 내용을 숨겨서 보내.
- 로그인(비밀번호)이나 긴 글쓰기처럼 보안이 필요하거나 서버의 상태를 바꿀 때 써.

---
## Practical Context (실무 활용)

현업에서는 이 두 가지 상황을 확실히 구분해야 에러가 안 나.

* **GET을 쓸 때:**
    * 네이버 뉴스 검색 (`query=파이썬`)
    * 오늘 날씨 데이터 가져오기
    * **특징:** 실행해도 서버 데이터가 변하지 않아 (Idempotent). 
    * [[Idempotent(멱등성)]] 참고 

* **POST를 쓸 때:**
    * 아이디/비번으로 로그인하기
    * 쇼핑몰 결제 버튼 누르기
    * 슬랙(Slack)에 봇으로 메시지 보내기
    * *특징:* 실행하면 서버에 새로운 데이터가 생기거나 변해.

---
##  Code Core Points

파이썬 `requests` 코드에서 가장 큰 차이는 **데이터를 실어 나르는 파라미터 이름**이야.

* **GET:** `params` = 딕셔너리 (URL 뒤에 꼬리표처럼 붙음)
* **POST:** `data` 또는 `json` = 딕셔너리 (몸통(Body)에 숨겨서 감)

---
##  Detailed Analysis

```python
import requests

# --- 1. GET 방식 (조회) ---
url_get = "https://httpbin.org/get"
payload = {'key1': 'value1', 'key2': 'value2'}

# GET은 'params' 인자를 사용해!
# 실제 요청 URL: [https://httpbin.org/get?key1=value1&key2=value2](https://httpbin.org/get?key1=value1&key2=value2)
r = requests.get(url_get, params=payload)

print(f"GET URL 확인: {r.url}") 
# 결과: 주소 뒤에 물음표(?) 달고 데이터가 다 보임.


# --- 2. POST 방식 (전송) ---
url_post = "https://httpbin.org/post"
payload_post = {'username': 'my_id', 'password': 'secret_pw'}

# POST는 'data' (Form 형식) 혹은 'json' (JSON 형식) 인자를 사용해!
r = requests.post(url_post, data=payload_post)

print(f"POST URL 확인: {r.url}")
# 결과: 주소 뒤에 아무것도 안 보임 ([https://httpbin.org/post](https://httpbin.org/post))
# 데이터는 r.request.body 안에 숨어있음.
print(r.json())
print(r.json()["form"])
```

---
## 초보자가 자주 착각하는 포인트

1. **"보안 때문에 POST 쓴다?"** 
	- 반은 맞고 반은 틀려. 
	- POST로 보내도 HTTPS(자물쇠)를 안 쓰면 해커가 중간에서 다 볼 수 있어. 
	- POST는 '안 보이는 척'하는 거지 암호화는 아니야.

2.  **"GET으로 긴 데이터 못 보내나?"** 
	- 응, 못 보내. 브라우저 주소창 길이 제한(약 2,000자)이 있어서 
	- 긴 글이나 파일 업로드는 무조건 POST여야 해.

3.  **인자 혼동:** 
	- `requests.get(url, data=...)`라고 쓴다고 에러는 안 나지만, 서버가 데이터를 못 찾아서 무시하는 경우가 많아. 
	- **GET은 params, POST는 data/json** 공식을 외워!