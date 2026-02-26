---
aliases:
  - streamlit
  - config.toml
  - 서버 배포 환경설정
  - Headless 모드
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Theming]]"
---
## `config.toml` : 내 앱의 성격과 집 주소를 정하는 법

---
## 한줄 요약 

`config.toml`은 코드로 짜인 앱(Content)에 겉모습(Theme)과 서버 구동 방식(Server)을 입혀주는 명세서이며, 특히 클라우드 배포 시 서버의 생존을 결정짓는 필수 설정 파일

.gitignore에 추가하는것이 아니다! 

---
## 왜 필요한가?

**문제점:** 
로컬(내 맥북)과 달리 클라우드 서버는 모니터나 키보드가 없는 '로봇'이야. 
만약 앱이 실행될 때 "이메일을 입력하세요(Welcome to Streamlit!)" 같은 사용자 입력 대기 창(프롬프트)을 띄우면, 서버는 아무것도 못하고 멈춰버려 결국 `Connection Refused` 에러를 뱉고 죽게 돼.

 **해결책:** 
 `headless=true` 설정을 명시해 주면, "여기는 입력해 줄 사람이 없으니 묻지 말고 백그라운드에서 조용히 앱만 실행해라!"라고 서버에게 강력하게 명령을 내릴 수 있어.

---
## 실무 맥락

로컬에서 예쁘게 만든 포트폴리오나 데이터 파이프라인 대시보드를 Streamlit Cloud, AWS EC2, 또는 Docker 컨테이너 같은 '리눅스 서버 환경'에 최초로 배포할 때 무조건 설정해야 하는 필수 관문이야.

---
## **Code Core Points:** 

프로젝트 최상단 `.streamlit/config.toml` 파일의 `[server]` 구역에 아래 코드를 추가하는 것이 핵심이야.

```TOML
[server]
# 클라우드 배포 시 선택이 아닌 필수! (대화형 프롬프트 비활성화)
headless = true

[browser]
# 불필요한 통계 수집으로 인한 리소스 낭비 방지
gatherUsageStats = false
```

---
## Detailed Analysis (설정 파일 완벽 해부)

### 설정 파일은 어디에 만들어야 할까? (Location)

Streamlit은 앱을 실행할 때 정해진 폴더들을 기웃거리며 설정 파일을 찾아.
실무에서는 다른 앱과 설정이 꼬이지 않도록 **프로젝트 레벨(Local) 설정**을 99% 사용해.

**프로젝트 레벨 (Local, ⭐ 추천):** 
현재 작업 중인 프로젝트 폴더 안에 숨김 폴더(`.streamlit`)를 만들고 그 안에 넣습니다. 
오직 이 프로젝트에만 룰이 적용됩니다.

- **경로:** `내_프로젝트_폴더/.streamlit/config.toml`

### `config.toml`의 4가지 핵심 구역 (Sections)

파일 안은 대괄호(`[ ]`)를 이용해 4개의 방으로 나뉘어 있어.

|**섹션 (구역)**|**역할**|**실무에서 가장 많이 건드리는 설정**|
|---|---|---|
|**`[theme]`**|앱의 톤앤매너, 색상, 폰트 등 **'디자인'** 담당|`primaryColor`, `backgroundColor`, `base`|
|**`[server]`**|포트 번호, 접속 등 **'네트워크와 통신'** 담당|`port`, `address`, **`headless`**|
|**`[browser]`**|사용자의 **'브라우저 화면 동작'** 제어|`gatherUsageStats` (통계 수집 끄기)|
|**`[client]`**|사용자 화면에 에러를 띄울지 등 제어|`showErrorDetails` (보안상 화면에서 에러 숨기기)|


### 누가 이길까? 설정의 "우선순위" (Priority)

포트 번호를 터미널 명령어로는 `8080`, 파일에는 `8501`로 적었다면? Streamlit은 언제나 **'가장 가깝고 직접적으로 내린 명령'** 을 최우선으로 따라.

1. **명령어 직접 입력 (CLI Flag):** `streamlit run app.py --server.port 8080` **(가장 강함 💪)**
2. **환경 변수 (Environment Variable):** `STREAMLIT_SERVER_PORT=8080`
3. **프로젝트 설정 (Local Config):** `.streamlit/config.toml`
4. **글로벌 설정 (Global Config):** `~/.streamlit/config.toml`
5. **Streamlit 기본값 (Default):** 아무것도 안 건드렸을 때의 상태 **(가장 약함 🏳️)**

> **💡 데이터 엔지니어의 실무 꿀팁:** 로컬 컴퓨터에서 예쁘게 꾸미고 테스트할 때는 **`config.toml`** 을 사용합니다. 
> 하지만 완성된 앱을 도커(Docker)로 배포할 때는, 파일은 그대로 둔 채 **환경 변수(`Environment Variable`)** 만 외부에서 주입하여 설정을 덮어씌우는 방식이 정석입니다.

---
## Common Beginner Misconceptions (초보자들이 흔히 하는 착각)

#### "`config.toml` 파일도 깃허브에 올라가면 안 되니까 `.gitignore`에 추가해야 하나요?"

**절대 NO!** API 키가 들어있는 `secrets.toml`은 무조건 숨겨야 하지만, `config.toml`은 서버가 앱을 어떻게 구동할지 알려주는 설명서야. 
서버가 이 파일을 읽을 수 있도록 **반드시 깃허브에 같이 Push 해야 해.**

#### "파일을 `app.py`랑 같은 폴더에 대충 둬도 알아서 읽겠지?"

**NO!** Streamlit은 무조건 프로젝트의 **최상단(Root)에 있는 `.streamlit` 폴더**만 찾아가.
폴더 경로가 하나라도 빗나가면 설정 파일 전체를 깡그리 무시해버려.
