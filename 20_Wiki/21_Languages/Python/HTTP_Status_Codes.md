---
aliases:
  - HTTP 상태 코드
  - Status Codes
  - 404 에러
  - 500 에러
  - 응답 코드
tags:
  - HTTP
  - Status_Code
related:
  - "[[Python_Requests_Response]]"
---
##  개념 한 줄 요약

**HTTP 상태 코드**는 서버가 내 요청을 처리한 결과를 3자리 숫자로 요약해준 **"성적표"** 혹은 **"신호등"** 이야.

---
## 왜 필요한가 (Why)

코드를 짤 때 가장 중요한 건 "누구 잘못인지" 빨리 파악하는 거야.

**문제점 (Without It):** 
- 에러가 났는데, 내가 주소를 틀린 건지(404), 서버가 터진 건지(500), 아니면 권한이 없는 건지(403) 모르면 수정할 방향을 못 잡아.

 **해결책:**
 -  앞자리 숫자(2, 4, 5)만 봐도 **"내 코드 수정"** 이 필요한지, 아니면 **"잠시 대기"** 가 필요한지 1초 만에 판단할 수 있어.

---
##  Practical Context (실무 필수 암기)

수십 개가 있지만, 데이터 엔지니어는 딱 이 3가지 그룹만 확실히 알면 돼.

### 🟢 2xx: 성공 (Success) -> "잘 됐어!"

* **200 (OK):** 가장 흔함. "요청 정상 처리 완료."
* **201 (Created):** "네가 보낸 데이터로 새로 무언가를 만들었어." (POST로 데이터 저장 성공 시 자주 보임).

### 🔴 4xx: 클라이언트 에러 (Client Error) -> "네 잘못이야 (내 코드 수정 필요)"

* **400 (Bad Request):** "요청 자체가 이상해." (파라미터 오타, 필수 값 누락 등).
* **401 (Unauthorized):** "누구세요?" (로그인 안 함, API 키 누락).
* **403 (Forbidden):** "아는 사람인 건 알겠는데, 넌 이거 못 봐." (권한 부족, 관리자 페이지 접근 등).
* **404 (Not Found):** "그런 거 없는데요." (URL 오타, 삭제된 게시물).
* **405 (Method Not Allowed):** "그 방식은 안 돼." (GET 써야 하는데 POST 썼을 때).

### 💥 5xx: 서버 에러 (Server Error) -> "서버 잘못이야 (기다리거나 연락 필요)"

* **500 (Internal Server Error):** "서버 코드 어딘가가 터졌어." (보통 개발자 실수).
* **502 (Bad Gateway):** "서버 연결 통로가 막혔어." (트래픽 폭주 등).
* **503 (Service Unavailable):** "서버 점검 중이거나 과부하 상태야."

---
##  Code Core Points

* **`response.status_code`**: 숫자를 직접 확인하는 가장 기본 방법.

* **`response.raise_for_status()`**: 
	* 200번대가 아니면 **강제로 에러(Exception)를 발생**시키는 메서드. 
	- 코드를 중단시키고 싶을 때 유용해.
---
##  Detailed Analysis

```python
import requests

url = "https://jsonplaceholder.typicode.com/todos/1)"
response = requests.get(url)

code = response.status_code

# 1. 가장 기초적인 분기 처리
if code == 200:
    print("✅ 성공! 데이터를 처리합니다.")
    data = response.json()

elif code == 404:
    print("❌ 실패: URL을 다시 확인하세요. 페이지가 없습니다.")

elif code == 403:
    print("⛔ 권한 없음: API 키가 맞는지 확인하세요.")

elif code >= 500:
    print("💥 서버 오류: 내 잘못 아님. 나중에 다시 시도(Retry) 하세요.")

else:
    print(f"❓ 기타 상태 코드 발생: {code}")


# 2. 실무 꿀팁 (raise_for_status 사용)
# 에러가 났을 때 if문으로 도배하기 싫다면?
try:
    response.raise_for_status() # 200번대가 아니면 여기서 즉시 에러 발생!
    print("성공했으니 다음 로직 진행")
except requests.exceptions.HTTPError as e:
    print(f"치명적인 에러 발생으로 중단: {e}")
```

---
## 초보자가 자주 착각하는 포인트

1.  **400번대 에러인데 재시도(Retry) 한다:**
    - 절대 안 돼! 400번대(내 잘못)는 **코드를 고쳐서** 다시 보내야지, 똑같은 코드로 계속 보내봤자 100번 다 실패해. 
    - 오히려 서버에서 차단당할 수도 있어.
    - 반면, **500번대(서버 잘못)는 잠시 후 재시도**하면 될 수도 있어.
        
2.  **200 OK = 원하는 데이터?**
    - 통신은 성공했지만, "검색 결과 없음(Empty List)"을 줄 수도 있어. 
    - 200이라고 무조건 데이터가 있다고 믿으면 안 돼. 
    - 데이터 내용물(`len(data) > 0`)도 확인해야 해.