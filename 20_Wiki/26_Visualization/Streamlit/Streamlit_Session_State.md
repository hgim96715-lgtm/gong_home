---
aliases:
  - Session State
  - 세션 스테이트
  - st.session_state
  - 상태 관리
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
---
# 상태 관리

## 개념 한 줄 요약 (Concept Summary)

**"앱이 새로고침(Rerun)되어도 변수 값이 날아가지 않게 붙잡아두는 '전역 저장소(Dictionary)'."** 

---
## 왜 쓰는가? (The Problem: Streamlit's Amnesia)

**Streamlit의 기본 동작: 무한 새로고침**

Streamlit은 버튼을 누르거나 입력을 할 때마다 **코드를 맨 위에서부터 다시 실행(Rerun)** 합니다.
* **문제점:** `count = 0`으로 변수를 만들고 버튼을 눌러 `count + 1`을 해도, 다시 실행되면서 `count`는 도로 `0`이 됩니다.

**해결책: `st.session_state`**
 
이곳에 저장된 값은 앱이 다시 실행되어도 **초기화되지 않고 유지(Persist)** 됩니다.
* **활용:** 카운터, 로그인 상태 유지, 장바구니 목록, 폼 입력 임시 저장 등.

---
## 핵심 문법 (Core Syntax)

`st.session_state`는 파이썬의 **딕셔너리(Dictionary)**와 완벽하게 똑같이 작동합니다.
점(`.`) 표기법과 대괄호(`[]`) 표기법 모두 사용 가능합니다.

### 초기화 (Initialization) - **가장 중요!**

앱이 실행될 때 딱 한 번만 변수를 만들어야 합니다. 
만약 그냥 `st.session_state.count = 0`이라고 쓰면, 리턴될 때마다 0으로 덮어씌워지므로 의미가 없습니다.

```python
# 1️⃣ 확인: "my_count"라는 이름표가 없니?
if "my_count" not in st.session_state:
    
    # 2️⃣ 생성: 그럼 "my_count"를 0으로 만들어줘! (여기 이름도 똑같아야 함!)
    st.session_state.my_count = 0 

# 3️⃣ 사용: 아까 만든 "my_count"를 1 증가시켜! (여기 이름도 똑같아야 함!)
if st.button("증가"):
    st.session_state.my_count += 1
```

#### **핵심 규칙:** `st.session_state.` 뒤에 붙는 이름(`count`)은 **변수명**과 같습니다. 

`apple`, `score`, `data_list` 등 **아무거나 원하는 이름으로 지어도 됩니다
단, **처음 만들 때(초기화) 쓴 이름과 나중에 불러올 때(사용) 쓰는 이름이 100% 똑같아야** 파이썬이 기억할 수 있습니다.


### 읽기 및 쓰기 (Access & Update)

딕셔너리 방식(`[]`)과 점 표기법(`.`) 둘 다 가능합니다.

```python
# 값 변경 (쓰기)
st.session_state.count += 1      # 점 표기법 
st.session_state["name"] = "Kim" # 딕셔너리 표기법 

# 값 출력 (읽기)
st.write(st.session_state.count)
```

---
## 실무 심화 테크닉 (Advanced Skills)

### A. 위젯의 `key` 파라미터와 자동 동기화 (마법)

버튼, 텍스트 입력창 같은 위젯에 `key` 파라미터를 주면, 그 이름으로 **자동으로 `st.session_state`에 저장되고 실시간으로 동기화**됩니다! (초기화 코드를 짤 필요가 없습니다.)

```python
# 1. key="search_word"를 주는 순간, st.session_state.search_word 가 자동 생성됨!
st.text_input("검색어 입력", key="search_word")

# 2. 사용자가 타이핑하는 즉시 아래 코드가 알아서 업데이트됨
st.write(f"당신이 검색한 단어: {st.session_state.search_word}")
```

### B. 에러 없는 안전한 조회: `.get()`

`st.session_state["선택한파일"]`로 바로 접근하면, 만약 그 변수가 아직 안 만들어졌을 때(앱 첫 실행 등) 화면에 시뻘건 `KeyError`가 나면서 앱이 터집니다. 이때 딕셔너리의 `.get()` 메서드를 쓰면, 값이 없을 때 에러 대신 `None`을 반환하므로 훨씬 안전합니다.

```python
# ❌ 하수: 키가 없으면 에러로 앱 튕김
# current_file = st.session_state["selected_md"] 

# ⭕ 고수: 키가 없으면 조용히 빈 문자열("")이나 None을 반환함
current_file = st.session_state.get("selected_md", "")
```

### C. Active UI 패턴 (선택된 버튼 색칠하기)

"내가 현재 무엇을 보고 있는지" 상태(State)를 기반으로 UI의 디자인(버튼 색상)을 동적으로 바꾸는 패턴입니다.

```python
# 1. 지금 그릴 버튼의 경로(f["path"])가, 세션에 저장된 현재 선택 경로와 똑같은지 검사
is_active = st.session_state.get("selected_md") == f["path"]

# 2. 맞으면 파란색(primary), 아니면 회색(secondary) 버튼 렌더링!
if st.button(f"📄 {f['name']}", key=f["path"], type="primary" if is_active else "secondary"):
    # 버튼이 눌리면 이 파일의 경로를 세션 메모리에 덮어쓰기!
    st.session_state["selected_md"] = f["path"]
```

---
## 초기화 및 삭제 (Clear & Delete)

특정 동작(예: 로그아웃, 검색 초기화)을 했을 때 메모리를 비워주는 방법입니다.

```python
# 1. 특정 변수 하나만 삭제할 때
if "search_word" in st.session_state:
    del st.session_state["search_word"]

# 2. 메모리 전체를 싹 다 날려버릴 때 (로그아웃 시 유용)
st.session_state.clear()
```


---
## 실전 예제: 클릭 카운터 (Counter Example)

버튼을 누를 때마다 숫자가 올라가는 가장 기본적인 예제입니다.

```python
import streamlit as st

# 1. 초기화 (필수)
if "clicks" not in st.session_state:
    st.session_state.clicks = 0

# 2. 버튼 클릭 시 값 증가
if st.button("클릭하세요"):
    st.session_state.clicks += 1  # 값 업데이트

# 3. 결과 출력
st.write(f"버튼을 {st.session_state.clicks}번 눌렀습니다.")
```

---
## Key Takeaways

- **Dictionary-like:** 파이썬 딕셔너리처럼 다루면 됩니다.
- **Persistence:** 리런(새로고침)이 되어도 데이터가 살아남습니다.
- **Interactive Apps:** 대시보드나 멀티 페이지 앱을 만들 때 필수적인 기능입니다