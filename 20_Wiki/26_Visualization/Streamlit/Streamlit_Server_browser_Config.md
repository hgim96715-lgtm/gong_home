---
aliases:
  - server
  - streamlit
  - port
  - address
  - headless
  - enableCORS
  - browser
  - gatherUsageStats
  - serverAddress
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Config]]"
  - "[[Streamlit_Theming]]"
  - "[[Streamlit_Layout]]"
---
## [server] 내 앱의 '집 주소'와 '출입문' 관리하기


## 왜 서버 설정을 따로 해줘야 할까?

Streamlit은 `streamlit run`을 치는 순간, 기본적으로 **"내 컴퓨터(localhost)의 8501번 방(포트)"** 에 앱을 띄우고 브라우저를 강제로 엽니다.

하지만 우리가 만든 앱을 도커 컨테이너에 가두거나, 클라우드(AWS EC2 등)에 올릴 때는 **"외부 사람이 어떻게 접속할지"**, 그리고 **"브라우저가 없는 서버 환경에서 어떻게 동작할지"** 를 `.streamlit/config.toml`의 `[server]` 구역에 명확히 적어줘야 합니다.

---
## 서버 통신을 위한 핵심 4대장

`[server]` 섹션에서 실무 엔지니어들이 가장 많이 건드리는 4가지 옵션입니다.

|**설정 옵션**|**역할 (비유)**|**실무 꿀팁 💡**|
|---|---|---|
|**`port`**|앱이 열리는 **'방 번호'**|기본값은 `8501`입니다. 여러 개의 Streamlit 앱을 동시에 띄워야 할 때 `8502`, `8503` 등으로 바꿔서 충돌을 막습니다.|
|**`address`**|문을 두드릴 수 있는 **'허가증'**|기본값은 `localhost`(나만 접속 가능)입니다. 외부망이나 도커 환경에서 외부 접속을 허용하려면 반드시 `"0.0.0.0"`으로 설정해야 합니다.|
|**`headless`**|브라우저 팝업 **'자동 실행 방지'**|서버나 도커에는 화면(모니터)이 없습니다! 이 값을 `true`로 해두지 않으면, Streamlit이 브라우저를 열려고 시도하다가 에러를 뱉고 뻗어버릴 수 있습니다.|
|**`enableCORS`**|외부 도메인 **'접근 보안(CORS)'**|내 앱이 아이프레임(iframe)에 심어지거나 복잡한 네트워크 환경에 있을 때, 보안 문제로 화면이 하얗게 뜰 수 있습니다. 배포 시 꼬이면 임시로 `false`를 줘서 우회하기도 합니다.|

---
## 실전 배포 레시피 (Copy & Paste)

상황에 맞게 아래의 코드를 `.streamlit/config.toml` 파일에 그대로 복사해서 사용하세요.

### Case A. "도커(Docker) & 클라우드 배포용" (가장 중요!)

서버나 컨테이너 환경에서 앱을 띄울 때 **무조건 들어가야 하는 표준 세팅**입니다. 외부 접속을 뻥 뚫어주고, 쓸데없는 브라우저 실행을 막습니다.

```TOML
[server]
port = 8501
address = "0.0.0.0"       # 중요! 외부 IP 접속 허용 (도커 필수)
headless = true           # 중요! 브라우저 자동 실행 끄기
enableCORS = false        # 보안 충돌 방지 (필요 시 true로 변경)

[browser]
gatherUsageStats = false  # Streamlit 본사로 데이터 수집 보내는 것 끄기
```

### Case B. "안전한 로컬 개발용"

오직 내 노트북에서만 개발하고 띄워볼 때 쓰는 세팅입니다. 남들은 내 아이피를 알아도 접속할 수 없습니다.

```TOML
[server]
port = 8501
address = "localhost"     # 오직 내 컴퓨터에서만 접속 가능 (보안)
headless = false          # 실행하자마자 브라우저 창 띄워주기
```

---
## 터미널(CLI) 명령어로 설정 덮어쓰기

`config.toml` 파일을 만들기 귀찮거나, **도커파일(Dockerfile)** 안에서 명령어를 실행해야 할 때는 아래처럼 깃발(Flag)을 꽂아 강제로 설정을 먹일 수 있습니다. (설정 우선순위 1위!)

```bash
# 도커 컨테이너가 켜질 때 실행되는 명령어 예시
streamlit run app.py --server.port=8501 --server.address=0.0.0.0
```


---
## `[browser]` : 브라우저 동작 제어 및 오해 잡기

"사용자의 웹 브라우저가 내 앱과 어떻게 상호작용할지 결정합니다. 
불필요한 통계 수집을 끄고, 브라우저 탭 설정에 대한 흔한 오해를 바로잡는 구역입니다."

|**설정 옵션**|**역할**|**실무 꿀팁 💡**|
|---|---|---|
|**`gatherUsageStats`**|Streamlit 본사로 **사용 통계 전송 여부**|기본값은 `true`입니다. 실무(사내망)나 포트폴리오 배포 시, 불필요한 외부 네트워크 통신을 막기 위해 **무조건 `false`로 끄는 것이 국룰**입니다.|
|**`serverAddress`**|브라우저가 접속할 **실제 도메인/IP**|리버스 프록시(Nginx 등) 뒤에 앱이 숨어있을 때 브라우저에게 "이 주소로 찾아와!"라고 알려주는 고급 설정입니다. 보통은 안 건드려도 됩니다.|

---
##  주의! 브라우저 '탭(Tab) 이름'과 '아이콘'은 여기서 못 바꿉니다!

초보자들이 가장 많이 하는 실수가 `config.toml`에서 브라우저 탭 이름을 바꾸려고 하는 것입니다.
**브라우저 탭 제목과 파비콘(아이콘)은 설정 파일이 아니라, 반드시 파이썬 코드 안에서 개별적으로 설정해야 합니다.**

틀린 방법 (`config.toml`에 적기 - 동작 안 함)

```TOML
[browser]
pageTitle = "My Portfolio" # 이런 명령어는 존재하지 않습니다!
```

✅ 맞는 방법 (`app.py` 파이썬 코드 최상단에 적기)

```python
import streamlit as st

# 무조건 다른 streamlit 명령어보다 가장 먼저 실행되어야 합니다!
st.set_page_config(
    page_title="Data Engineer Portfolio", # 브라우저 탭 이름
    page_icon="🚀",                       # 브라우저 탭 아이콘(파비콘)
    layout="wide"                         # 화면 넓게 쓰기
)
```

>[[Streamlit_Layout]] 참고 
