---
aliases:
  - st.write
  - st.markdown
  - st.success
  - st.error
  - st.info
  - st.warning
  - st.toast
  - unsafe_allow_html=True
  - st.html
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Concept]]"
  - "[[Streamlit_Widgets]]"
  - "[[Streamlit_Layout]]"
  - "[[Streamlit_Advanced_CSS]]"
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

### ② 만능 출력 (`write`) ⭐

귀찮으면 그냥 `st.write()` 쓰세요. 
글자, 데이터프레임, 차트, 심지어 함수까지 알아서 보여줍니다.

```python
st.write("그냥 글자도 되고")
st.write(pd.DataFrame({'a':[1,2]})) # 표도 그려줌
```

---
## 마크다운 (`markdown`) & HTML 커스터마이징

마크다운 문법으로 글자 색상, 굵기, 링크 등을 꾸밀 수 있습니다.
하지만 **포트폴리오처럼 디테일한 디자인(중앙 정렬, 특정 색상 폰트)이 필요할 때는 HTML과 CSS를 혼합해서 사용**해야 합니다.


> 더 자세한 내용은 👉 [[Streamlit_Advanced_CSS]] 참고 ! 
> **Streamlit 1.33.0 버전 이상**부터는 HTML 전용 렌더링 함수인 **`st.html()`** 이 새롭게 추가 이게 더 좋음! 

**주의: Streamlit은 HTML을 기본적으로 무시합니다!**
아래 코드를 그대로 쓰면 화면에 태그가 텍스트처럼 그냥 노출되어 버립니다.

```python
# ❌ 실패: 화면에 "<h2>Data Engineer</h2>" 라고 문자 그대로 뜸
st.markdown("<h2>Data Engineer</h2>")
```

### ✅ 올바른 방법 1: `unsafe_allow_html=True` 사용하기 (디자인 커스텀)

HTML 태그를 브라우저가 직접 해석하게 만들려면 이 옵션이 반드시 필요합니다.
`config.toml`의 테마만으로는 부족한 **정렬이나 특정 포인트 컬러**를 줄 때 사용합니다.

```python
# 👉 단순 HTML 태그 적용
st.markdown("<h2>Data Engineer</h2>", unsafe_allow_html=True)

# 🎨 폰트 색상과 굵기 디테일 커스텀 (CSS 인라인 적용)
st.markdown(
    """
    <h2 style="color:#00ADB5; font-weight:700;">
        Data Engineer
    </h2>
    """,
    unsafe_allow_html=True
)

# ↔️ 텍스트 중앙 정렬
st.markdown(
    "<h2 style='text-align:center;'>Data Engineer</h2>",
    unsafe_allow_html=True
)
```


### ✅ 올바른 방법 2: Streamlit 기본 API 쓰기 (권장)

굳이 세밀한 CSS 커스텀이 필요한 게 아니라면, 기본 함수를 쓰는 것이 코드가 훨씬 깔끔하고 보안상 안전합니다.

```python
st.header("Data Engineer")      # HTML의 <h2>와 동일
st.subheader("Data Engineer")   # HTML의 <h3>와 동일
st.markdown("## Data Engineer") # 마크다운 문법 사용
```

### 언제 어떤 걸 쓰는 게 좋을까? (Decision Table)

|**상황**|**추천 방법**|**코드 예시**|
|---|---|---|
|**단순한 제목 작성**|Streamlit 내장 함수|`st.header("제목")`|
|**긴 문서 / 설명글**|마크다운 문법|`st.markdown("## 제목\n본문")`|
|**색상 / 폰트 / 정렬 커스텀**|**HTML + CSS 혼합**|`st.markdown("<h1 style='color:red;'>...</h1>", unsafe_allow_html=True)`|
|**포트폴리오 전체 브랜딩**|**config.toml 테마 활용**|`primaryColor` 및 `base` 설정 변경|


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

---
## 진행 상태 표시하기 (`st.spinner`)

네트워크 통신(API 호출), 무거운 데이터 로딩, 머신러닝 모델 추론 등 **오래 걸리는 작업**을 할 때 사용자를 안심시키는 '로딩 애니메이션'입니다.

- **원리:** `with` 문과 함께 사용합니다. `with` 블록 안의 코드(데이터 불러오기 등)가 실행되는 동안 빙글빙글 도는 스피너가 나타나고, **블록 안의 코드가 전부 끝나면 스피너는 마법처럼 자동으로 사라집니다.**

```python
import streamlit as st
import time

# 실무 활용 예시: 버튼을 누르면 로딩 스피너 작동
if st.button("데이터 불러오기"):
    # with 블록이 시작되는 순간 스피너 등장!
    with st.spinner("서버에서 대용량 데이터를 가져오는 중입니다... 🏃‍♂️💨"):
        
        time.sleep(3) # 엄청 오래 걸리는 작업이라고 가정
        data = "데이터 로딩 완료!" 
        
    # with 블록이 끝났으므로 스피너는 자동으로 사라짐
    st.success(data)
```
