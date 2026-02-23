---
aliases:
  - html
  - css
  - data-testid
  - Streamlit UI 커스텀
  - st.html
  - st.markdown
  - "mailto:"
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Layout]]"
  - "[[Streamlit_Status_Text]]"
  - "[[Streamlit_Theming]]"
---
## 한줄 요약

"config.toml만으로 해결되지 않는 디테일한 UI(사이드바 여백, 못생긴 구분선, 외부 아이콘)를 HTML/CSS 주입으로 완벽하게 통제하는 실전 프론트엔드 해킹 기법."


---
## 핵심 개념: `data-testid`가 도대체 뭔데?

Streamlit 앱을 띄우고 크롬 브라우저에서 **개발자 도구(F12)** 를 열어보면, HTML 클래스 이름들이 `.st-emotion-cache-1wmy9hl` 처럼 해괴한 문자열로 되어 있습니다.

**문제점:** 
이 클래스 이름은 Streamlit 버전이 업데이트되거나 앱을 다시 실행할 때마다 **랜덤하게 바뀝니다.** 여기에 CSS 스타일을 먹이면 며칠 뒤에 디자인이 다 깨집니다!

**구원자 `data-testid`:** 
Streamlit 개발팀이 자동화 테스트를 하려고 HTML 태그마다 **"절대 변하지 않는 고정 이름표"** 를 붙여두었는데, 이게 바로 `data-testid`입니다.

- 예: 사이드바 영역 = `data-testid="stSidebar"`
- 예: 메인 화면 영역 = `data-testid="stAppViewBlockContainer"`

**활용:**  고정된 이름표를 CSS 스나이퍼 소총의 조준점으로 삼아서 스타일을 정확하게 꽂아 넣습니다.

---
## 궁극의 CSS 방어술: `:has` 선택자로 랜덤 클래스명 제압하기 (Advanced)

위에서 배운 `data-testid`만으로 해결되지 않는 경우가 있습니다. 바로 **"랜덤한 이름을 가진 부모 컨테이너"** 의 크기나 여백을 조절해야 할 때입니다.

이때 최신 CSS 문법인 **`:has()` 가상 클래스**를 쓰면, 부모의 이름이 무엇이든 상관없이 "특정 자식(`data-testid`)을 품고 있는 부모"의 멱살을 잡아 통제할 수 있습니다!

#### 실무 예제: 버튼 크기를 줄바꿈에 상관없이 칼같이 고정하기

```css
/* ❌ 하수: 랜덤 이름으로 찾기 (내일 되면 깨짐) */
.st-emotion-cache-zh2fnc { height: 100px; }

/* ⭕ 고수: :has()로 찾기 (평생 안 깨짐) */
/* 해석: "내 자식 중에 data-testid가 'stButton'인 놈이 있다면, 부모인 내 높이를 무조건 100px로 고정해라!" */
div:has(> div[data-testid="stButton"]) {
    width: 100% !important;
    height: 100px !important;
}
```


---
## 최신 CSS 주입 스킬 (`st.html`의 압승)

과거에는 HTML이나 CSS를 넣으려면 무조건 `st.markdown("<style>...</style>", unsafe_allow_html=True)`를 써야만 했습니다. 
하지만 이 과거 방식은 치명적인 단점이 있었는데, 바로 **코드 상에 보이지 않는 '투명한 빈 줄(여백)'을 만들어내서 전체 레이아웃을 틀어지게 한다는 것**이었습니다.

**Streamlit 1.33.0 버전**부터 도입된 **`st.html()`**은 이 문제를 완벽하게 해결한 구원자입니다!

**[실무 황금 공식]** "CSS, 외부 CDN 링크, 커스텀 HTML은 이제 무조건 `st.html()` 하나로 통일한다!"

- **왜 `st.markdown`을 버렸나요?** `st.html()`은 코드 안에 `<style>` 태그만 있다는 것을 감지하면, 화면의 공간을 전혀 차지하지 않는 '이벤트 컨테이너(Event Container)' 영역으로 알아서 숨어버립니다. 덕분에 불필요한 여백 없이 깔끔한 UI를 유지할 수 있습니다.
    
- **직관적인 코드:** 길고 귀찮은 `unsafe_allow_html=True` 옵션을 안 써도 되기 때문에 코드가 훨씬 짧고 직관적입니다.

```python
import streamlit as st

# 이제 markdown이 아니라 html로 한 번에 깔끔하게 주입합니다!
st.html("""
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
    <style>
        /* 앱 전체에 적용될 공통 디자인 룰 */
        section[data-testid="stSidebar"] { padding-top: 2rem !important; }
        section[data-testid="stSidebar"] a.link-btn { text-decoration: none; }
    </style>
""")
```


---
## 나중에 써먹기 위한 CSS 속성(Attribute) 사전

|**CSS 속성**|**역할 (무엇을 바꾸는가)**|**포트폴리오 실무 팁 💡**|
|---|---|---|
|**`padding`**|박스 **안쪽** 여백|`padding-top: 2rem;` (위쪽 안쪽 여백을 2rem만큼 줌). 사이드바 상단이 너무 텅 비었을 때 줄이는 용도로 씁니다.|
|**`margin`**|박스 **바깥쪽** 여백|`margin: 1.5rem 0;` (위아래 1.5rem, 좌우 0). 내가 만든 가로선(Divider) 위아래 간격을 띄울 때 씁니다.|
|**`border`**|박스의 **테두리 선**|`border-top: 1px solid #30363d;` (위쪽에만 1픽셀짜리 실선을 다크그레이 색상으로 그음). 못생긴 기본 구분선 대신 씁니다.|
|**`text-decoration`**|텍스트 **밑줄** 등 꾸밈|`text-decoration: none;` (링크 텍스트의 촌스러운 파란 밑줄을 없애버림).|
|**`display`**|요소의 **배치 방식**|`display: inline-block;` (텍스트를 버튼처럼 네모난 블록 형태로 취급해서 여백을 먹일 수 있게 함).|
|**`font-family`**|**글꼴** 지정|`font-family: monospace;` (코딩할 때 쓰는 타자기 폰트를 적용해 개발자 느낌을 줌).|

---
## Font Awesome 아이콘

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
```

### 1단계: Streamlit에 도서관(CDN) 연결하기

앱 최상단(페이지 설정 직후)에 `st.markdown`을 이용해 CDN과 CSS를 한방에 뚫어줍니다. 
이렇게 해야 사이드바든 메인 화면이든 어디서든 아이콘을 불러올 수 있습니다.(이건 앱 전체에서 딱 한 번만 하면 됩니다.)

```python
import streamlit as st

# 🚨 전역 CSS 및 아이콘 CDN은 반드시 markdown(unsafe_allow_html=True)으로 선언!
st.markdown("""
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
    <style>
        /* 아까 배운 사이드바 여백 줄이기, 커스텀 선 등 전역 CSS도 여기에 같이 넣습니다 */
        section[data-testid="stSidebar"] a.link-btn { text-decoration: none; }
    </style>
""", unsafe_allow_html=True)
```

### 2단계: 원하는 아이콘 찾기

[Font Awesome 공식 홈페이지](https://fontawesome.com/search?o=r&m=free)에 접속합니다.
검색창에 영어로 원하는 키워드를 칩니다. (예: `github`, `database`, `server`)
마음에 드는 아이콘을 클릭하면 `<i class="fa-solid fa-database"></i>` 같은 HTML 코드가 나옵니다.

```python
# ===== 실제 사용 예시 (사이드바 내부) =====
with st.sidebar:
    # 이제 st.html만 써도 아이콘이 기가 막히게 잘 뜹니다!
    st.html('<a href="https://github.com" class="link-btn"><i class="fa-brands fa-github"></i> GitHub</a>')
    
    # 일반 텍스트에 섞어 쓸 때
    st.html('<i class="fa-solid fa-server"></i> <b>Kafka Pipeline Status</b>')
```

### 데이터 엔지니어 전용 치트시트 (복붙용)

| **용도 / 의미**        | **Font Awesome HTML 코드**                   | **설명**                         |
| ------------------ | ------------------------------------------ | ------------------------------ |
| **GitHub**         | `<i class="fa-brands fa-github"></i>`      | 소스코드, 깃허브 링크 연결 시              |
| **LinkedIn**       | `<i class="fa-brands fa-linkedin"></i>`    | 링크드인, 이력서 연결 시                 |
| **이메일(Contact)**   | `<i class="fa-solid fa-envelope"></i>`     | 연락처 표시                         |
| **DB / Data Lake** | `<i class="fa-solid fa-database"></i>`     | PostgreSQL, MinIO 등 저장소 표시     |
| **서버 / 파이프라인**     | `<i class="fa-solid fa-server"></i>`       | Kafka, Spark, 인프라 아키텍처 표시      |
| **차트 / 대시보드**      | `<i class="fa-solid fa-chart-simple"></i>` | 데이터 시각화 결과물 표시                 |
| **톱니바퀴 (설정/배치)**   | `<i class="fa-solid fa-gear"></i>`         | Airflow, Batch Job 등 돌아가는 프로세스 |


---
## 클릭하면 메일 앱이 열리는 'Contact' 버튼 만들기 (`mailto:`)

포트폴리오 사이드바에 GitHub, LinkedIn 링크와 함께 **"나에게 메일 보내기"** 버튼을 만들 때 사용하는 가장 우아한 방법입니다.

일반적인 `https://` 웹 링크 대신, `href="mailto:이메일주소"`를 사용하면 방문자가 버튼을 클릭했을 때 기기에 설치된 기본 메일 프로그램이 자동으로 실행되며 받는 사람 주소가 세팅됩니다.

```python
# ===== 사이드바 Contact Me 섹션 예시 =====

with st.sidebar:
    st.subheader("👨‍💻 Contact Me")
    
    # 1. 일반 웹사이트 링크 (새 창 열기: target="_blank")
    st.html('<a href="https://github.com/my-profile" class="link-btn" target="_blank"><i class="fa-brands fa-github"></i> GitHub</a>')
    st.html('<a href="https://linkedin.com/in/my-profile" class="link-btn" target="_blank"><i class="fa-brands fa-linkedin"></i> LinkedIn</a>')
    
    # 2. 이메일 전송 링크 (mailto: 사용) 
    # 주의: mailto는 새 창을 띄우는 게 아니므로 target="_blank"를 뺍니다!
    st.html('<a href="mailto:hgim96715@email.com" class="link-btn btn-email"><i class="fa-solid fa-envelope"></i> Email Send</a>')
```

HTML의 `mailto:` 속성 뒤에 파라미터(`?`와 `&`)를 붙여서 메일의 제목(Subject)과 본문(Body)을 미리 채워둘 수 있습니다.

>**주의사항: URL 인코딩 (URL Encoding)**
>웹 브라우저는 띄어쓰기나 줄바꿈을 그대로 인식하지 못하므로, 반드시 **URL 인코딩 규칙**을 따라야 합니다.
>**공백 (띄어쓰기):** `%20`
>**줄바꿈 (Enter):** `%0A`

**제목만 미리 채우기 (`?subject=`)**

```python
# 💡 띄어쓰기 대신 %20을 넣어야 에러가 나지 않습니다.
st.html("""
    <a href="mailto:hgim96715@email.com?subject=Portfolio%20Inquiry" class="link-btn btn-email">
        <i class="fa-solid fa-envelope"></i> 문의 메일 보내기
    </a>
""")
```

제목 + 본문 첫 줄까지 세팅하기 (`&body=`)

```python
st.html("""
    <a href="mailto:hgim96715@email.com?subject=Portfolio%20Inquiry&body=Hello%20Kim%20Han%20Gyeong," class="link-btn btn-email">
        <i class="fa-solid fa-envelope"></i> Contact Me
    </a>
""")
```

**줄바꿈(`%0A`)이 포함된 완벽한 템플릿**

```python
# 미리보기:
# 제목: [포트폴리오 문의] 데이터 엔지니어 포지션
# 본문: 
# 안녕하세요 김한경 님,
# 
# (여기에 내용을 입력하세요)

st.html("""
    <a href="mailto:hgim96715@email.com?subject=[포트폴리오%20문의]%20데이터%20엔지니어%20포지션&body=안녕하세요%20김한경%20님,%0A%0A(여기에%20내용을%20입력하세요)" class="link-btn btn-email">
        <i class="fa-solid fa-envelope"></i> Email Send
    </a>
""")
```







