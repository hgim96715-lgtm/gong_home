---
aliases:
  - Multipage App
  - 멀티페이지
  - Streamlit Pages
  - 사이드바 메뉴
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"메인 파일(`Home.py`)과 `pages/` 폴더만 있으면, 자동으로 사이드바 메뉴가 생기는 웹사이트 구조."** 

---

## 왜 쓰는가? (Why)

앱의 기능이 많아지면(예: 데이터 탐색, 차트, 설정 등) 한 페이지에 다 담을 수 없습니다.
Streamlit의 멀티페이지 기능을 쓰면 **웹사이트의 탭(Tab)처럼 섹션을 나눌 수 있어** 대규모 앱을 체계적으로 관리할 수 있습니다. 

---

## 프로젝트 구조 (Project Structure)

가장 중요한 것은 **폴더 구조**입니다.
Streamlit은 `pages`라는 이름의 폴더를 자동으로 인식합니다. 

```text
my_app/
├── Home.py              # 메인 진입 페이지 (Landing Page) 
└── pages/               # 추가 페이지들은 무조건 이 폴더 안에! 
    ├── 1_Data_Explorer.py
    ├── 2_Charts.py
    └── 3_About.py
```

### 파일명 규칙 (Naming Convention)

- **`Home.py`**: 앱의 대문 역할을 합니다.
- **`pages/` 폴더**: 이 안에 있는 `.py` 파일 하나가 웹페이지 하나가 됩니다.
- **숫자 접두사 (`1_`, `2_`)**: 사이드바 메뉴의 **순서(Order)** 를 결정합니다.
    - 예: `1_Data.py`가 `2_Chart.py`보다 위에 뜹니다.

---
## 실행 방법 (How to Run)

`pages` 폴더 안의 파일이 아닌, **프로젝트 최상위 경로(Root)** 에 있는 **메인 파일**을 실행해야 합니다.

```bash
# 메인 파일명이 Home.py인 경우
streamlit run Home.py

# 메인 파일명이 Main.py인 경우
streamlit run Main.py
```

실행하면 왼쪽 사이드바에 **메인 파일의 이름**이 첫 번째 메뉴로 생성됩니다.

- **Home** (또는 Main, App 등 설정한 파일명)
- Data Explorer
- Charts
- About

**⚠️ 주의 (Naming Rules)**

1. **메인 파일(`Home.py`)**: 이름은 `App.py`, `Main.py` 등 자유롭게 바꿔도 됩니다. 
> 단, 이름을 바꿨다면 실행할 때 반드시 **`streamlit run {바꾼이름}.py`** 라고 입력해야 합니다.

2. **`pages/` 폴더**: 이 폴더의 이름은 **절대 바꾸면 안 됩니다.** Streamlit이 `pages`라는 이름을 찾아 자동으로 메뉴를 구성하기 때문입니다.

---
## 핵심 팁 (Tips & Tricks)

1. **데이터 공유 (Sharing Data):** 페이지가 바뀌면 변수가 초기화됩니다. 페이지끼리 데이터를 넘겨주려면(예: 1페이지에서 로그인 → 2페이지에서 이름 출력) 반드시 **`st.session_state`**  사용해야 합니다.
2. **1 Page = 1 Feature:** 유지보수를 위해 한 페이지에는 하나의 핵심 기능만 담는 것이 좋습니다.
3. **라이브러리 독립성:** 각 페이지(`py` 파일)는 서로 다른 라이브러리나 레이아웃을 가질 수 있어 자유도가 높습니다.

