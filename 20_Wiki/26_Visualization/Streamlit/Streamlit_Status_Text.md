---
aliases:
  - st.write
  - st.markdown
  - st.success
  - st.error
  - st.info
  - st.warning
  - st.toast
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Concept]]"
  - "[[Streamlit_Widgets]]"
  - "[[Streamlit_Layout]]"
---
## 개념 한 줄 요약

**"단순한 텍스트 출력부터, 사용자의 시선을 끄는 예쁜 알림창(Alert)까지 한 방에 해결."**

---
##  텍스트 출력의 기본 (`title`, `header`, `write`)

Streamlit에서 글자를 보여주는 방법은 다양합니다.

### ① 계층 구조 (Hierarchy)

```python
import streamlit as st

st.title("1. 가장 큰 제목 (Title)")
st.header("2. 중간 제목 (Header)")
st.subheader("3. 소제목 (Subheader)")
st.text("4. 그냥 텍스트 (폰트 스타일 없음)")
```

---
### ② 마크다운 (`markdown`)

글자 색상, 굵기, 링크 등을 자유롭게 꾸밀 때 사용합니다.

```python
st.markdown("이것은 **굵은 글씨**이고, 이것은 [구글](https://google.com) 링크입니다.")
st.markdown("---") # 가로선 긋기
```

### ③ 만능 출력 (`write`) ⭐

귀찮으면 그냥 `st.write()` 쓰세요. 
글자, 데이터프레임, 차트, 심지어 함수까지 알아서 보여줍니다.

```python
st.write("그냥 글자도 되고")
st.write(pd.DataFrame({'a':[1,2]})) # 표도 그려줌
```

---
## 상태 알림창 (Status Messages)

사용자에게 **"성공했다", "실패했다", "주의해라"** 메시지를 직관적인 색상 박스로 보여줍니다.

### ① 4가지 색깔 박스

가장 많이 쓰는 기능입니다.

```python
# 🟢 초록색 (성공)
st.success("✅ 데이터 저장에 성공했습니다!")

# 🔵 파란색 (정보)
st.info("ℹ️ 현재 버전은 v1.0.2 입니다.")

# 🟡 노란색 (경고)
st.warning("⚠️ 파일 용량이 큽니다. 시간이 걸릴 수 있습니다.")

# 🔴 빨간색 (에러)
st.error("🚫 연결 실패! 서버 상태를 확인하세요.")
```

### ② 예외 처리 (`exception`)

파이썬 에러(Traceback)를 화면에 그대로 보여줄 때 씁니다. 디버깅용으로 좋습니다.

```python
try:
    1 / 0
except Exception as e:
    st.exception(e) # 에러 로그가 예쁘게 출력됨
```

----
## 사라지는 메시지 (`toast`)

모바일 앱 알림처럼 우측 하단에 **잠깐 떴다가 사라지는** 메시지입니다. 
화면 공간을 차지하지 않아서 깔끔합니다.

```python
import time

if st.button("저장하기"):
    st.toast("💾 저장되었습니다!", icon="✅")
    time.sleep(1)
```

---
## 실전 꿀팁: 빈 의자(`empty`)와 함께 쓰기

알림창은 보통 **"처리 중..."** 이었다가 **"성공!"** 으로 바뀌어야 제맛이죠.
이때 `st.empty()`와 `status` 메시지를 결합하면 완벽합니다.

```python
import time

# 1. 빈 공간 확보
status_box = st.empty()

# 2. 정보 표시 (파란색)
status_box.info("⏳ 데이터를 처리하는 중입니다...")
time.sleep(2)

# 3. 성공 표시로 교체 (초록색)
status_box.success("✅ 처리가 완료되었습니다!")
```

