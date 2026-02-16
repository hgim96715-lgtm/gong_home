---
aliases:
  - Streamlit Layout
  - 컬럼
  - 탭
  - 사이드바
  - Expander
  - Page Config
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Widgets]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"위젯을 가로로 배치하거나(Columns), 숨기거나(Expander), 탭으로 묶어서(Tabs) 공간을 효율적으로 쓰는 기술."**

---
##  페이지 기본 설정 (Page Config) 

스크립트 **가장 맨 위**에 작성해야 하는 필수 설정입니다.
앱의 제목, 아이콘, 그리고 전체 레이아웃 모드를 결정합니다.

```python
import streamlit as st

st.set_page_config(
    page_title="실시간 대시보드",
    page_icon="📊",
    layout="wide",                         # "centered"(기본) vs "wide"(꽉 찬 화면)
    initial_sidebar_state="expanded"       # 사이드바 열림 상태 
)
```

- **`layout="wide"`**: 대시보드를 만들 때 필수입니다. 브라우저 가로폭을 꽉 채워 씁니다.
- **`initial_sidebar_state`**: 앱 시작 시 사이드바를 열어둘지(`expanded`) 닫아둘지(`collapsed`) 정합니다.

---
## 화면 분할 (Layout Components)

Streamlit은 콘텐츠를 구조화하기 위한 내장 레이아웃 도구를 제공합니다.

### ① 컬럼 (`st.columns`) - 가로 배치

화면을 N개로 쪼개서 가로로 나란히 배치합니다. 
차트 2개를 양옆에 띄울 때 씁니다.

```python
col1, col2 = st.columns(2)  # 1:1 비율로 분할

with col1:
    st.header("왼쪽 구역")
    st.line_chart(data1)

with col2:
    st.header("오른쪽 구역")
    st.bar_chart(data2)
```

#### 💡 꿀팁: 비율 내 맘대로 조절하기 (`[2, 1]`)

괄호 안에 숫자가 아니라 **리스트(`[]`)** 를 넣으면 각 컬럼의 너비 비율이 됩니다. 
메인 차트를 크게 보여주고, 설명글을 작게 보여줄 때 필수입니다.

```python
# 왼쪽이 2배 더 넓음 (2:1 비율)
main_col, side_col = st.columns([2, 1])

with main_col:
    st.write("여기는 넓은 메인 화면 (차트)")

with side_col:
    st.write("여기는 좁은 사이드 (설명)")
```

**(중앙 정렬):** `st.columns([1, 2, 1])` 처럼 양옆을 비우고 가운데만 크게 잡으면, 컴포넌트를 **화면 정중앙**에 예쁘게 배치할 수 있습니다.



### ② 탭 (`st.tabs`) - 겹쳐서 배치

내용이 많을 때, 브라우저 탭처럼 클릭해서 전환하게 만듭니다. 
공간 절약에 최고입니다.

```python
tab1, tab2 = st.tabs(["매출 데이터", "Raw 로그"])

with tab1:
    st.write("매출 차트가 들어갑니다.")
with tab2:
    st.write("복잡한 로그 데이터는 여기서 보세요.")
```

### ③ 익스팬더 (`st.expander`) - 접었다 펴기

중요하지 않거나 자리를 많이 차지하는 내용(예: 에러 로그, 도움말)을 숨겨둘 때 씁니다.

```python
with st.expander("ℹ️ 상세 로그 확인하기"):
    st.write("여기에 엄청 긴 로그 데이터가 들어갑니다...")
```

### ④ 사이드바 (`st.sidebar`) - 고정 메뉴

화면 왼쪽에 고정된 패널입니다.
주로 네비게이션 메뉴나 필터 설정(날짜 선택 등)을 넣습니다.

```python
st.sidebar.title("설정")
date = st.sidebar.date_input("날짜 선택")
```

---
## 시각적 구분 (Spacing & Dividers)

내용 사이를 깔끔하게 나누고 싶을 때 사용합니다.

- **`st.divider()`**: 가로선(Horizontal Line)을 그어줍니다. > `st.markdown("---")` 과 같은 역할 
- **`st.write("")`**: 빈 줄을 넣어 여백을 줍니다

