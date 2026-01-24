---
aliases:
  - Requests Response 객체
  - Response Object
  - Python 응답 객체
tags:
  - Python
  - Requests
  - HTTP
related:
  - "[[Python_JSON]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[Airflow_Hooks]]"
  - "[[Python_Requests_Methods]]"
---
##  개념 한 줄 요약

**Requests Response 객체**는 우리가 서버에 요청(Request)을 보낸 뒤, "서버가 편지 봉투에 담아서 돌려준 답장 그 자체"라고 보면 돼.

**Content-Type**은 서버가 우리에게 데이터를 보낼 때 "내가 지금 보낸 데이터는 이런 형식(타입)이니까 거기에 맞춰서 해석해!"라고 알려주는 **데이터의 신분증** 같은 거야.

---
## 왜 필요한가 (Why)

**문제점 (Without It):**
* 초보 때는 그냥 `response.json()`이나 `response.text`만 외워서 쓰곤 해. 
* 근데 이러면 서버가 404 에러(페이지 없음)나 500 에러(서버 터짐)를 보냈을 때도 무작정 데이터를 꺼내려다가 프로그램이 죽어버려. 
* "편지가 왔는데 내용물도 안 보고 봉투부터 찢는 격"이지.

**해결책:** 
- Response 객체는 단순히 데이터만 들고 있는 게 아니라, **상태 코드(Status Code), 인코딩 방식, 헤더 정보** 같은 "배송 정보"를 다 갖고 있어. 
- 이걸 알면 에러 처리를 튼튼하게 할 수 있고, 글자가 깨지는(인코딩) 문제도 해결할 수 있어.

**문제점 (Without It):** 
* 서버가 주는 데이터는 그냥 0과 1로 된 이진 데이터(Binary)일 뿐이야. 
* 이게 텍스트인지, 이미지인지, 아니면 JSON인지 모르면 파이썬은 이걸 어떻게 처리해야 할지 몰라 헤매게 돼.

**해결책:**
- `Content-Type`을 확인하면 우리가 `.json()`을 써야 할지, 아니면 파일로 저장해야 할지 등을 명확하게 결정할 수 있어. 
- 잘못된 방식으로 파싱하다가 에러(Runtime Error)가 나는 걸 방지해 주지.

---
## Practical Context (실무 활용)

현업 데이터 엔지니어링에서 데이터를 수집(크롤링/API 호출)할 때 가장 많이 마주치는 객체야.

* **언제 쓰나:** `requests.get()`이나 `requests.post()`를 실행하고 나면 반환되는 게 바로 Response 객체야.
* **어떻게 쓰나:**
    1.  먼저 `status_code`를 확인해서 통신이 성공했는지 본다.
    2.  성공했으면 `.json()`이나 `.text`로 알맹이를 꺼낸다.
    3.  가끔 이미지가 필요하면 `.content`로 바이너리 데이터를 꺼낸다.

**`application/json`**: API 응답의 표준. 파이썬 딕셔너리로 바꿀 수 있는 데이터야. (json 데이터)
**`text/html`**: 일반적인 웹 페이지. 크롤링(BeautifulSoup 등)이 필요할 때 보여.
**`image/jpeg` 또는 `image/png`**: 이미지 데이터. `.text`가 아니라 `.content`로 받아야 해.

> "응답(Response)을 받기 전에 요청(Request)을 보내는 방법(GET/POST)이 헷갈린다면 [[Python_Requests_Methods]] 문서를 참고할 것."

---
## Code Core Points 

코드를 보기 전에 이 흐름을 먼저 머리에 넣어둬.

1.  **Request 실행:** `requests.get()`을 하면 서버로 달려갔다 옴.
2.  **객체 수신:** 결과물이 `Response` 클래스의 인스턴스로 돌아옴 (변수명은 보통 `res`, `resp`, `r` 등을 씀).
3.  **검증:** `.ok` 혹은 `.status_code`로 봉투가 잘 도착했는지 확인.
4.  **추출:** 내용물 형식에 맞춰 꺼냄 (`.text` vs `.json()` vs `.content`).
5.  **딕셔너리처럼 접근:** `response.headers`는 딕셔너리와 비슷하게 생겨서 키(Key)값으로 불러와.
6. **대소문자 무관:** `requests` 라이브러리가 똑똑해서 `Content-Type`이나 `content-type`이나 똑같이 찾아줘.
7. **세부 정보 포함:** 단순히 타입만 주는 게 아니라 인코딩(charset) 정보도 같이 들어있는 경우가 많아.

---
## Detailed Analysis


```python
import requests

# 1. 요청 보내기
url = "[https://jsonplaceholder.typicode.com/todos/1](https://jsonplaceholder.typicode.com/todos/1)"
response = requests.get(url) 

# --- 여기서부터가 Response 객체 분석 ---

# 2. 통신 상태 확인 (가장 중요!)
# 200이면 성공, 404나 500이면 실패.
print(f"상태 코드: {response.status_code}") 

# 3. 성공 여부를 Boolean으로 쉽게 확인
# status_code가 200~299 사이면 True, 아니면 False
if response.ok:
    print("통신 성공! 데이터를 까봅니다.")
    
    # 4. 헤더 확인 (메타 데이터)
    # 서버가 보내준 데이터가 JSON인지 HTML인지, 언제 보냈는지 등이 들어있음
    content_type = response.headers.get("Content-Type", "")
    print(f"데이터 타입: {content_type}")
    
    # 5. Content-Type에 따른 처리 분기
    if "application/json" in content_type:
    # 데이터 추출 (JSON인 경우)
    # .json()은 메서드(함수)임에 주의! 괄호 () 필수.
    # Response 객체 내부의 텍스트를 파이썬 딕셔너리로 변환해줌.
        data = response.json()
        print("✅ JSON 데이터입니다.")
        print(f"할 일 제목: {data['title']}")

    elif "text/html" in content_type:
        print("📄 HTML 문서입니다.")
        print(response.text[:300])  # 앞부분만 출력

    elif "image" in content_type:
        print("🖼 이미지 파일입니다.")
        with open("image.jpg", "wb") as f:
            f.write(response.content)
        print("image.jpg 파일로 저장했습니다.")

    else:
        print("❓ 알 수 없는 타입입니다.")
        print(response.text[:300])

else:
    # 6. 에러 처리
    # 에러가 났을 때 무작정 파싱하면 코드가 터짐.
    print("문제가 생겼습니다.")
    # 보통 여기서 재시도를 하거나 로그를 남김
    print(f"에러 내용(본문): {response.text}")
```

---
## 초보자가 자주 착각하는 포인트


1. **`response`는 그냥 데이터가 아니다:**
    - 초보자들이 `print(response)`를 하면 데이터가 출력될 줄 아는데, `<Response [200]>`이라는 객체 정보만 나와. 
    - 실제 내용은 `.text`나 `.content` 안에 들어있어.

2. **`.text` vs `.json()`:**
    - `.text`는 **속성(변수)** 이라 괄호가 없고, 그냥 '글자 덩어리(String)'를 줘.
    - `.json()`은 **메서드(함수)** 라 괄호 `()`가 필요하고, 파이썬의 '딕셔너리/리스트'로 변환해서 줘. 
    - JSON 형식이 아닌데 이걸 쓰면 에러 나.

3. **인코딩 문제:**
    - 한글이 깨져서 보일 때가 있어. 그럴 땐 당황하지 말고 `response.encoding = 'utf-8'` 처럼 인코딩을 수동으로 지정해주고 나서 `.text`를 찍어보면 해결될 때가 많아.

4. **`Accept`와 헷갈리지 마:**
	- `Accept`: 우리가 서버한테 "나 JSON으로 **받고 싶어**"라고 요청하는 것 (Request Header).
	- `Content-Type`: 서버가 우리한테 "자, 이건 JSON **데이터야**"라고 주는 것 (Response Header).

5. 세미콜론(`;`) 뒷부분:
	- `application/json; charset=utf-8` 처럼 뒤에 뭐가 붙어 있는 경우가 많아. 그래서 `{text}==`으로 비교하기보다는 `in` 연산자를 써서 키워드가 포함되어 있는지 확인하는 게 훨씬 안전해.

6. **항상 믿을 수는 없다:**
	- 간혹 서버 설정 문제로 실제 데이터는 JSON인데 `text/plain`으로 보내주는 불친절한 서버도 있어. 
	- 이럴 땐 우리가 직접 데이터를 보고 판단해야 하는 데이터 엔지니어의 '직감'이 필요하지!

7. **참고 :**
	- `application/octet-stream`이라고 온다면, 그건 "무슨 데이터인지 정의할 수 없는 이진 데이터"라는 뜻이야.

---
## 참고 사이트

### jsonplaceholder

- https://jsonplaceholder.typicode.com/
- API 연습용으로 가장 유명한 사이트

### 예시 도메인

- https://example.com
- 크롤링 연습용 표준 사이트

### **HTTP 테스트 전용 사이트** (실무에서도 자주 씀)

- https://httpbin.org/
- HTTP / 상태코드 / 이미지