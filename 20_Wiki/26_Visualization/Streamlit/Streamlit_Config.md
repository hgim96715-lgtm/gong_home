---
aliases:
  - streamlit
  - config.toml
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Theming]]"
---
## `config.toml` : 내 앱의 성격과 집 주소를 정하는 법

> **"코드가 앱의 내용(Content)이라면, `config.toml`은 앱의 겉모습(Theme)과 거주지(Server)를 결정하는 명세서입니다."**

---
## 설정 파일은 어디에 만들어야 할까? (Location)

Streamlit은 앱을 실행할 때 두 군데의 폴더를 기웃거리며 설정 파일을 찾습니다. 
실무에서는 다른 앱과 꼬이지 않도록 **프로젝트 레벨(Local) 설정**을 99% 사용합니다.

**프로젝트 레벨 (Local, ⭐ 추천):** 
현재 작업 중인 프로젝트 폴더 안에 숨김 폴더(`.streamlit`)를 만들고 그 안에 넣습니다. 
오직 이 프로젝트에만 룰이 적용됩니다.

- **경로:** `내_프로젝트_폴더/.streamlit/config.toml`

**글로벌 레벨 (Global):** 
내 컴퓨터(또는 서버)에서 돌아가는 모든 Streamlit 앱에 공통으로 강제 적용할 때 씁니다.

**경로:** `~/.streamlit/config.toml` (Mac/Linux) 또는 `%USERPROFILE%\.streamlit\config.toml` (Windows)

---
### 누가 이길까? 설정의 "우선순위" (Priority)

만약 터미널 명령어로는 포트를 `8080`으로 주라고 하고, `config.toml` 파일에는 `8501`로 적혀있다면 앱은 어디로 뜰까요? Streamlit은 언제나 **'가장 가깝고 직접적으로 내린 명령'**을 최우선으로 따릅니다.

1. **명령어 직접 입력 (CLI Flag):** `streamlit run app.py --server.port 8080` **(가장 강함 💪)**
2. **환경 변수 (Environment Variable):** `STREAMLIT_SERVER_PORT=8080`
3. **프로젝트 설정 (Local Config):** `.streamlit/config.toml`
4. **글로벌 설정 (Global Config):** `~/.streamlit/config.toml`
5. **Streamlit 기본값 (Default):** 아무것도 안 건드렸을 때의 상태 **(가장 약함 🏳️)**

> **💡 데이터 엔지니어의 실무 꿀팁:** 로컬 컴퓨터에서 예쁘게 꾸미고 테스트할 때는 **`config.toml`** 을 사용합니다. 
> 하지만 완성된 앱을 도커(Docker)로 배포할 때는, 파일은 그대로 둔 채 **환경 변수(`Environment Variable`)** 만 외부에서 주입하여 설정을 덮어씌우는 방식이 정석입니다.

---
### `config.toml`의 4가지 핵심 구역 (Sections)

파일 안은 대괄호(`[ ]`)를 이용해 4개의 방으로 나뉘어 있습니다. 
변경하고 싶은 속성에 맞춰 알맞은 방에 값을 적어주면 됩니다.

| **섹션 (구역)**     | **역할**                           | **실무에서 가장 많이 건드리는 설정**                    |
| --------------- | -------------------------------- | ----------------------------------------- |
| **`[theme]`**   | 앱의 톤앤매너, 색상, 폰트 등 **'디자인'** 담당   | `primaryColor`, `backgroundColor`, `base` |
| **`[server]`**  | 포트 번호, 접속 허용 등 **'네트워크와 통신'** 담당 | `port`, `address`, `headless`             |
| **`[browser]`** | 사용자의 **'브라우저 화면 동작'** 제어         | `gatherUsageStats` (불필요한 통계 수집 끄기)        |
| **`[client]`**  | 사용자 화면에 에러를 띄울지 등 **'클라이언트'** 설정 | `showErrorDetails` (보안상 화면에서 에러 숨기기)      |