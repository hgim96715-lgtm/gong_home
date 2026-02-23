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
  - "[[Streamlit_Advanced_CSS]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Streamlit_Dynamic_UI]]"
---
# 화면 분할과 공간 배치의 모든 것

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

- **비율 분할:** `main_col, side_col = st.columns([2, 1])` (왼쪽이 2배 더 넓음)
- **중앙 정렬 꼼수:** `st.columns([1, 2, 1])` 처럼 양옆을 비우고 가운데만 크게 잡으면, 컴포넌트를 **화면 정중앙**에 예쁘게 배치할 수 있습니다.

```python
# 왼쪽이 2배 더 넓음 (2:1 비율)
main_col, side_col = st.columns([2, 1])

with main_col:
    st.write("여기는 넓은 메인 화면 (차트)")

with side_col:
    st.write("여기는 좁은 사이드 (설명)")
```

### 탭 (`st.tabs`) - 겹쳐서 배치

내용이 많을 때, 브라우저 탭처럼 클릭해서 전환하게 만듭니다. 
공간 절약에 최고입니다.

##### **핵심 원리 (가장 많이 하는 착각):** 

`st.tabs(["이름1", "이름2"])`는 단순히 화면에 예쁜 탭을 그리기만 하는 게 아닙니다
인자로 넣은 이름의 개수만큼 **'데이터를 담을 수 있는 텅 빈 공간(컨테이너 객체)'들을 만들어서 리스트(List) 형태로 반환**합니다.
우리가 `with` 문을 쓸 수 있는 이유도, 이 반환된 '공간' 안으로 직접 들어가서 그림을 그리기 때문입니다.

#### **A. 기본 사용법 (탭 개수가 고정되어 있을 때)**

```python
tab1, tab2 = st.tabs(["매출 데이터", "Raw 로그"])

with tab1:
    st.write("매출 차트가 들어갑니다.")
with tab2:
    st.write("복잡한 로그 데이터는 여기서 보세요.")
```

#### B. 🌟 실무 고급 사용법 (탭 개수가 유동적일 때)

카테고리 목록처럼 개수가 계속 변하는 리스트를 던져줄 때 쓰는 치트키입니다.

```python
import streamlit as st

subs = ["Python", "SQL", "Docker"]

# 3개의 탭 이름이 담긴 리스트를 던지면, 3개의 '탭 공간 객체' 리스트가 반환됨!
tabs = st.tabs(subs) 

# zip()을 이용해 '이름'과 '공간'을 지퍼처럼 1:1로 짝지어서 반복문 돌리기
for name, tab_space in zip(subs, tabs):
    with tab_space: # 각각의 탭 공간 안으로 순서대로 진입!
        st.write(f"여기는 {name} 카테고리의 학습 노트입니다.")
```


### 익스팬더 (`st.expander`) - 접었다 펴기(아코디언)

중요하지 않거나 자리를 너무 많이 차지하는 내용(예: 에러 로그, 도움말, 세부 설정)을 숨겨둘 때 씁니다.

```python
with st.expander("ℹ️ 상세 로그 확인하기", expanded=False): # expanded=True면 기본으로 열려있음
    st.write("여기에 엄청 긴 로그 데이터가 들어갑니다...")
```

### ④ 사이드바 (`st.sidebar`) - 고정 메뉴

화면 왼쪽에 고정된 패널입니다.
주로 네비게이션 메뉴나 필터 설정(날짜 선택 등)을 넣습니다.

```python
st.sidebar.title("설정")
date = st.sidebar.date_input("날짜 선택")
```

### ⑤ 컨테이너 (`st.container`) - 투명 박스 묶기

화면에 시각적인 테두리는 없지만, 여러 요소들을 하나의 '논리적인 그룹(박스)'으로 묶어줄 때 사용합니다.

```python
# st.tabs와 구조적 일관성을 맞추기 위해 '가짜 탭(투명 박스)'을 만들 때도 유용합니다.
empty_box = st.container()

with empty_box:
    st.write("이 내용은 투명 박스 안에 그룹화되어 출력됩니다.")
```

---
## 시각적 구분 (Spacing & Dividers)

내용 사이를 깔끔하게 나누고 싶을 때 사용합니다.

- **`st.divider()`**: 가로선(Horizontal Line)을 그어줍니다. > `st.markdown("---")` 과 같은 역할 
- **`st.write("")`**: 빈 줄을 넣어 여백을 줍니다

