---
aliases:
  - Streamlit 부분 새로고침
  - st.fragment
  - 화면깜빡임 방지
tags:
  - Streamlit
related:
  - "[[Streamlit_Cache]]"
  - "[[Streamlit_Forms]]"
  - "[[Streamlit_Session_State]]"
  - "[[00_Streamlit_HomePage]]"
---
# 전체 새로고침 방지! 딱 이 구역만 다시 그리기

## 개념 한 줄 요약 (Concept Summary)

"버튼 하나 눌렀다고 화면 전체가 깜빡거리며 처음부터 끝까지 재실행되는 걸 막아주는 '독립된 격리 구역' 만들기!"

---
## 왜 쓰는가? (The Problem: Full App Rerun)

**🔥 Streamlit의 태생적 한계** Streamlit은 사용자가 버튼을 누르거나 글자를 입력할 때마다 **코드를 맨 위에서부터 아래로 통째로 다시 실행(Rerun)** 합니다.

- **문제점:** 무거운 머신러닝 모델을 로딩하거나 대용량 DB 쿼리를 돌려놨는데, 고작 '검색어' 하나 입력했다고 화면 전체가 새로고침되면서 로딩이 또 걸립니다. 화면이 하얗게 깜빡거리는 건 덤이죠!

**💡 해결책: `@st.fragment` (부분 새로고침)** 특정 UI 구역(함수)에 `@st.fragment` 장식을 달아주면, **그 구역 안에서 일어나는 클릭이나 입력은 앱 전체를 재실행시키지 않고 오직 '그 함수 안쪽'만 다시 그립니다.**

---
## 핵심 문법 (Core Syntax)

독립적으로 움직여도 되는 UI 덩어리를 **함수(`def`)** 로 묶고, 그 위에 `@st.fragment` 데코레이터를 붙여주기만 하면 끝입니다.

```python
import streamlit as st
import time

st.title("메인 화면 (전체 새로고침 영역)")
st.write(f"전체 화면 렌더링 시간: {time.strftime('%H:%M:%S')}")

# 👇 이 구역 안에서 버튼을 눌러도, 위쪽의 '전체 화면 렌더링 시간'은 바뀌지 않습니다!
@st.fragment
def independent_counter():
    st.subheader("독립된 카운터 구역")
    
    if "count" not in st.session_state:
        st.session_state.count = 0
        
    if st.button("프래그먼트 내부 버튼"):
        st.session_state.count += 1
        
    st.write(f"현재 카운트: {st.session_state.count}")
    st.write(f"부분 렌더링 시간: {time.strftime('%H:%M:%S')}")

# 함수 호출하여 화면에 그리기
independent_counter()
```

---
## ## 실무 주의사항: 프래그먼트의 덫 (Fragment Constraints)

> **"내가 허락한 공간 밖에는 그림을 그릴 수 없다!"**

`StreamlitFragmentWidgetsNotAllowedOutsideError` 에러를 만났던 바로 그 이유
- **원칙:** 프래그먼트 함수는 **"자신이 호출된 바로 그 공간"** 안에서만 위젯을 그릴 수 있습니다.
-  **에러 발생 원인:** 함수 밖에서 만든 탭이나 빈 박스(`st.container`)를 함수 안으로 끌고 들어와서, 억지로 그 바깥 공간에 위젯을 밀어 넣으려 하면 Streamlit 엔진이 보안 및 렌더링 충돌로 에러를 뱉어냅니다.

**❌  (에러 발생 패턴): 함수 안에서 밖으로 손을 뻗음**

```python
empty_box = st.container()

@st.fragment
def bad_fragment(box):
    with box: # 🚨 에러! 프래그먼트 안에서 외부 컨테이너에 위젯을 그리려고 함!
        st.button("클릭")

bad_fragment(empty_box)
```

⭕  (완벽한 패턴): 밖에서 공간을 열고 그 안에서 함수를 부름

```python
empty_box = st.container()

@st.fragment
def good_fragment():
    # 자연스럽게 자기가 호출된 곳에 그어짐
    st.button("클릭") 

with empty_box: # ✅ 밖에서 미리 공간을 열어둠
    good_fragment() # 공간 안으로 들어가서 안전하게 렌더링!
```

---
## 꿀팁: 스스로 갱신하는 '실시간 모니터링' 대시보드 만들기

`@st.fragment`에는 `run_every`라는 엄청난 숨겨진 파라미터가 있습니다. 
사용자가 아무것도 안 눌러도, **지정된 시간(초)마다 알아서 그 구역만 새로고침**하여 최신 데이터를 가져옵니다! (실시간 로그 모니터링이나 주식 차트에 필수)

```python
# 5초마다 이 구역만 알아서 새로고침 됩니다! (전체 화면 깜빡임 없음)
@st.fragment(run_every=5)
def realtime_log_viewer():
    st.subheader("🔥 실시간 서버 로그")
    # 최신 로그를 DB에서 긁어오는 가상의 함수
    logs = fetch_latest_logs() 
    st.dataframe(logs)

realtime_log_viewer()
```

