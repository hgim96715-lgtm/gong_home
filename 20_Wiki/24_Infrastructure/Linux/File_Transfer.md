---
aliases:
  - curl
  - wget
  - 파일다운로드
  - API요청
  - 데이터수집
tags:
  - Linux
related:
  - "[[Network_Status]]"
  - "[[00_Linux_HomePage]]"
---
## 1개념 한 줄 요약

**"터미널 세상의 '웹 브라우저'이자 '다운로드 매니저'."**

* **`wget` (World Wide Web Get):** "무조건 다운로드!"에 집중한 도구. (파일 내려받기 전문)
* **`curl` (Client URL):** "서버와 대화하기"에 집중한 도구. (API 요청, 데이터 전송 전문)

---

## 왜 필요한가? (Why)

**문제점:**
- 리눅스 서버에는 크롬(Chrome)이나 엣지(Edge) 같은 브라우저가 없다.
- "공공데이터 포털에서 CSV 파일을 받아야 하는데, 마우스 클릭을 할 수가 없네?"
- "내가 만든 API가 잘 작동하는지 테스트해보고 싶은데, 코드를 다 짜서 돌려봐야 하나?"

**해결책:**
- **`wget`** 으로 인터넷 주소(URL)만 알면 대용량 파일도 안정적으로 내려받는다.
- **`curl`** 로 터미널에서 바로 서버에 데이터를 보내거나(`POST`) 가져와서(`GET`) 테스트한다.

---

## 실무 적용 사례 (Practical Context)

데이터 엔지니어는 이 두 가지를 명확히 구분해서 씁니다.

1.  **데이터셋 다운로드 (`wget`):**
    - "캐글(Kaggle)이나 공공데이터에서 10GB짜리 학습 데이터를 서버로 가져올 때."
    - 인터넷이 끊겨도 이어받기(`-c`)가 강력해서 `wget`을 씀.

2.  **API 연동 및 테스트 (`curl`):**
    - "슬랙(Slack)으로 알림 메시지를 보내는 봇을 테스트할 때."
    - "Elasticsearch에 데이터를 집어넣거나 조회할 때."
    - 복잡한 헤더(Header)나 데이터(Body)를 같이 보내야 해서 `curl`을 씀.

---
## Code Core Points: ① `wget` (다운로드 특화) 
 
사용법이 아주 단순하다. 그냥 주소만 주면 받는다.

`wget [옵션] [옵션의 값(저장할 파일명)] 실제 다운로드 할 URL`

```bash
# 1. 기본 다운로드
# URL에 있는 파일명을 그대로 사용해 현재 디렉터리에 저장
wget https://example.com/data.csv


# 2. 끊긴 다운로드 이어받기 (-c)
# 대용량 파일 다운로드 중 네트워크가 끊겨도
# 같은 명령을 다시 실행하면 중단된 지점부터 이어서 받음
wget -c https://example.com/huge_dataset.zip


# 3. 이름 바꿔서 저장하기 (-O : 대문자 O)
# - 서버 파일명이 의미 없거나 너무 길 때
# - 쿼리스트링(?id=123)이 붙은 URL일 때
# - 스크립트/파이프라인에서 관리하기 쉬운 이름으로 저장하고 싶을 때
# ⚠️ 같은 이름의 파일이 있으면 기존 파일을 덮어씀
# 형식: wget -O [저장할_파일명] [URL]
wget -O my_data.csv https://example.com/weird_name_123.csv


# 4. 폴더/사이트 통째로 재귀 다운로드 (-r)
# 링크된 파일들을 따라가며 디렉터리 구조 그대로 다운로드
# (웹 크롤링과 유사한 동작)
wget -r https://example.com/directory/


# 5. 백그라운드에서 다운로드 (-b)
# 터미널을 닫아도 다운로드가 계속 진행됨
# 대용량 파일을 퇴근 전에 걸어둘 때 사용
# 진행 로그는 'wget-log' 파일에 저장됨
wget -b https://example.com/huge_dataset.zip
```

|옵션|의미|실무 포인트|
|---|---|---|
|`-c`|이어받기|대용량 파일 필수|
|`-O`|저장 파일명 지정|쿼리 URL 대응|
|`-r`|재귀 다운로드|미러링/크롤링|
|`-b`|백그라운드|배치/야간 작업|


---
## Code Core Points: ② `curl` (만능 칼) 

`curl`은 기본적으로 **결과를 화면(Standard Output)에 뿌린다.** 
파일로 저장하려면 옵션이 필요하다.

### A. 파일 다운로드

```bash
# 1. 화면에 출력 (파일로 저장하지 않음)
# API 테스트나 응답 내용 빠르게 확인할 때 사용
curl https://example.com/data.json

# 2. 파일로 저장하기 (-o : 소문자 o) 
# 내가 원하는 파일명으로 저장
# 형식: curl -o [저장할_파일명] [URL]
curl -o result.json https://example.com/data.json

# 3. 원본 파일명 그대로 저장 (-O : 대문자 O)
# 서버에서 내려주는 파일명을 그대로 사용
# URL의 마지막 경로가 파일명이 됨
curl -O https://example.com/data.json

# 4. 리다이렉트 따라가기 (-L) 
# 301 / 302 리다이렉트가 걸려 있으면
# 최종 목적지까지 자동으로 따라가서 다운로드
# 이 옵션 없으면 0바이트 파일 받는 경우 많음
curl -L -O https://example.com/moved_file.zip
```

### B. API 요청 (데이터 엔지니어 필수)

```bash
# 1. GET 요청 (기본값)
# 서버에서 데이터 조회
# -X GET 은 생략 가능 (curl의 기본 동작)
curl https://api.example.com/users


# 2. POST 요청 (폼 데이터 전송)
# HTML form 처럼 key=value 형식 데이터 전송
# -d 옵션을 쓰면 자동으로 POST 요청이 됨
curl -X POST https://example.com/form \
     -d "name=John&age=30"

# 3. POST 요청 (JSON 데이터 전송)
# 로그인, 데이터 생성/저장 API에서 가장 흔한 형태
# -H : HTTP 헤더 설정
# -d : 전송할 JSON 데이터
curl -X POST https://api.example.com/login \
     -H "Content-Type: application/json" \
     -d '{"id":"admin", "pw":"1234"}'
     
# 4. 헤더(Header) 추가하기 (-H) 
# 인증 토큰, Content-Type 설정 등에 사용
curl https://example.com/api/users \
     -H "Authorization: Bearer <my_secret_token>" \
     -H "Content-Type: application/json"
```

| 옵션             | 의미       | 설명                     |
| -------------- | -------- | ---------------------- |
| `-X`           | HTTP 메서드 | GET, POST, PUT, DELETE |
| `-d`           | 데이터 전송   | 사용 시 자동으로 POST         |
| `-H`           | 헤더 추가    | 인증, JSON 지정            |
| `Content-Type` | 데이터 형식   | JSON 사용 시 필수           |



---

## 상세 분석: `wget` vs `curl`

|**특징**|**wget**|**curl**|
|---|---|---|
|**주특기**|**안정적인 다운로드** (이어받기, 재귀)|**데이터 전송** (API, Upload)|
|**재귀 다운로드**|가능 (`-r`: 폴더째로 긁기)|불가능 (파일 하나씩만)|
|**프로토콜**|HTTP, HTTPS, FTP|HTTP, FTP, SCP, LDAP 등 20+개|
|**기본 동작**|파일로 바로 저장|화면에 내용 출력 (Stdout)|

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`curl` 쳤는데 화면에 글자만 막 쏟아져요!"

- `curl`은 기본적으로 **"화면 출력"** 이 목적입니다.
- 파일로 만들려면 꼭 **`-o`** 옵션이나 **`>`** 리다이렉션을 써야 합니다.

### ② "`wget`으로 받았는데 파일명이 이상해요."

- `wget`은 URL의 마지막 부분을 파일명으로 씁니다. 
- 만약 URL이 `.../download?id=123` 처럼 생겼으면 파일명이 `download?id=123`이 될 수 있습니다.
- 이때는 **`-O` (대문자)** 옵션으로 이름을 지정해 주세요.

### ③ "API 테스트하는데 401 Unauthorized가 떠요."

- 십중팔구 **헤더(`-H`)** 에 인증 토큰을 안 넣어서 그렇습니다. 
- `curl -H "Authorization: ..."` 부분을 꼭 확인하세요!



