
>[!source]
>페이지 [Streamlit 공식 홈페이지](https://streamlit.io/)
>portfolio [나의 portfolio](https://gongstudyde-ixkmtnttnimw8wszmtxgzg.streamlit.app/)



>Streamlit에서 실시간으로 숫자가 바뀌는 원리는 **"빈 의자(Placeholder)를 먼저 놓고, 나중에 사람(데이터)을 앉히는 방식"** 

## 화면 구성하기 (Basic UI)

> **"일단 눈에 보이는 것을 배치해보자."**

- [[Streamlit_Concept]] : Streamlit의 특징과 Vibe Coding, 빈의자 전략 (`placeholder`)
- **[[Streamlit_QuickStart]]** :  설치(`pip`) 및 실행(`python3 -m`) 완벽 가이드
- **[[Streamlit_Status_Text]]** : 제목, 본문, 그리고 알림창 (`success`, `error`, `info`, `markdown`,`spinner`)
- [[Streamlit_Widgets]] : 버튼, 슬라이더, 텍스트 입력 등 기본 입출력
- **[[Streamlit_Layout]]** : 컬럼(`col`), 탭(`tab`), 사이드바(`sidebar`)로 화면 배치하기
- [[Streamlit_Dynamic_UI]] : 파이썬으로 HTML/Markdown 붕어빵 찍어내기

## 데이터 그리기 (Visualization) 

> **"숫자를 그림으로 바꿔보자. 대시보드의 꽃!"**

- **[[Streamlit_Metrics]]** : 핵심 지표(KPI)와 변화량(Delta) 보여주기 (`st.metric`,`delta_color`,`delta`)
- **[[Streamlit_Charts]]** :  내장 차트(Simple) vs 외부 라이브러리(Powerful)
- **[[Streamlit_Data_Display]] : 웹 화면에 데이터프레임 출력하기 (`st.dataframe`,`use_container_width`,`hide_index`)** 
- **[[Streamlit_Mermaid]] : 마크다운 속 다이어그램을 웹에 그리기 (`streamlit-mermaid`와 파싱 기법)** 


## 데이터 주고받기 (Data I/O) 

>**"내 파일을 올리고, 분석 결과를 다운로드 받자."**

- **[[Streamlit_File_IO]]** :  `file_uploader`와 `download_button` 완벽 정리


## 동작 제어하기 (Flow & Logic)

> **"앱이 멍청하게 깜빡거리지 않고, 똑똑하게 기억하게 만들자." **

- [[Streamlit_Forms]] : 입력을 모아서 한 번에 전송 (최적화)
- [[Streamlit_Session_State]] : 새로고침 되어도 데이터 기억하기 (메모리) **+ 안전한 상태 조회(`.get`)와 Active UI 패턴**
- **[[Streamlit_Control_Flow]] :  밑으로 내려가지 마! 실행 흐름 강제 차단하기 (`st.stop`, `st.rerun`)** 


## 보안 및 설정 (Security & Config) 

> **"비밀번호, API 키를 코드에 적지 마세요! 해킹 당합니다."** 

- **[[Streamlit_Secrets]]** : `secrets.toml`로 민감한 정보 안전하게 숨기기


## 스타일 및 서버 설정 (Configuration)

> **"내 앱의 테마와 서버 설정을 내 입맛대로! 옷을 입히고 집 주소를 정하자."**

- **[[Streamlit_Config]]** : `.streamlit/config.toml` 파일의 위치와 설정 우선순위 이해하기
- **[[Streamlit_Theming]]** : 브랜드 컬러 입히기 (`primaryColor`, `backgroundColor`, `font` 설정)
- [[Streamlit_Advanced_CSS]] : CSS 주입 및 외부 아이콘(CDN) 연동(font-awesome), UI 강제 덮어쓰기 (`data-testid`),`st.html`,`st.markdown`,`mailto:`
- **[[Streamlit_Server_Browser_Config]]** : 서버 접속(`port`, `address`), 보안(`CORS`, `headless`) 및 브라우저 최적화(`gatherUsageStats`) 설정



## 앱 확장하기 (Architecture)

> **"기능이 많아지면 방을 나누자. 파일 하나에서 탈출하기!"**

- **[[Streamlit_Multipage]]** :  `pages` 폴더를 이용해 여러 페이지 만들기


## 성능 최적화 (Optimization) 

> **"매번 로딩하지 말고 기억하자! 속도 10배 높이기."** 

- **[[Streamlit_Cache]]** : `@st.cache_data`와 `@st.cache_resource`로 중복 연산 방지,`ttl`
- **[[Streamlit_Fragment]] : 전체 새로고침 방지! 딱 이 구역만 다시 그리기 (`@st.fragment`)**

## 실무 치트키 (Cheat Codes)

> **"Streamlit의 한계를 부수는 파이썬 외부 라이브러리 3대장"**

- **[[Streamlit_External_Libs]]** : 외부 데이터와 고급 UI를 위한 필수 모듈 (`requests`, `base64`, `re`)