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

`st.session_state`는 파이썬의 **딕셔너리(Dictionary)** 처럼 작동합니다.

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