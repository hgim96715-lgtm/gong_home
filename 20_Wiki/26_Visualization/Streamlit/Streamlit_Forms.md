---
aliases:
  - Streamlit Form
  - st.form
  - 입력 폼
  - 배치 제출
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"모든 입력을 다 채울 때까지 기다렸다가, '제출(Submit)' 버튼을 누르면 한 번에 실행하는 컨테이너."** 

---
## 왜 쓰는가? (The Problem & Solution)

 **기본 동작의 문제점 (Instant Update)**

Streamlit은 기본적으로 위젯 값이 변경될 때마다 앱 전체를 **즉시 다시 실행(Rerun)** 합니다.
* **문제:** 사용자가 검색창에 "S", "Se", "Seo", "Seoul"을 칠 때마다 쿼리가 4번 날아갑니다. (비용/속도 낭비)

 **해결책: `st.form`**

입력값들을 폼(Form)으로 묶어두면, **제출 버튼을 누르기 전까지는 앱이 다시 실행되지 않습니다.** 
* **효과:** 사용자가 모든 필드를 채우고 "조회"를 눌렀을 때 딱 한 번만 쿼리를 실행합니다.

---
## 기본 문법 (Basic Syntax)

`with` 문을 사용하여 폼 영역을 지정합니다. 
가장 중요한 규칙은 **반드시 `st.form_submit_button`으로 끝나야 한다**는 점입니다. 

**시작**: `with st.form("Key (고유 식별자):폼 이름"):`으로 문을 엽니다.

>**규칙**: form안에 적는 이름은 폼을 구별하는 이름표 . 페이지 내에서 **유일한(Unique)** 이름이어야 합니다

**내용**: 들여쓰기(Indent) 후 입력 위젯(`text_input`, `slider` 등)을 배치합니다.
**끝**: 반드시 **`st.form_submit_button("버튼명")`** 으로 마무리를 지어야 폼이 완성됩니다


```python
import streamlit as st

# 1. 폼 시작 (고유한 key 필수)
with st.form("my_form"): 
    st.write("사용자 정보를 입력하세요")
    
    # 2. 입력 위젯 배치
    name = st.text_input("이름") 
    age = st.slider("나이", 0, 100, 25) 
    
    # 3. 제출 버튼 (필수!)
    # 이 버튼을 눌러야만 위의 name, age 값이 확정됨
    submitted = st.form_submit_button("전송") 

# 4. 제출 후 동작
if submitted: 
    st.write(f"반갑습니다, {name}님! ({age}세)")
```

---
## 고급 활용: 한 페이지에 여러 폼 (Multiple Forms)

한 화면에 '로그인 폼'과 '피드백 폼'을 동시에 띄울 수도 있습니다. 
이때는 **`key`** 값을 다르게 주어야 충돌하지 않습니다.

```python
col1, col2 = st.columns(2)

# 왼쪽: 로그인 폼
with col1:
    with st.form("login_form"): # key="login_form"
        user = st.text_input("ID")
        pw = st.text_input("PW", type="password") [cite: 131]
        if st.form_submit_button("로그인"):
            st.success(f"{user}님 환영합니다!")

# 오른쪽: 피드백 폼
with col2:
    with st.form("feedback_form"): # key="feedback_form"
        msg = st.text_area("건의사항")
        if st.form_submit_button("보내기"):
            st.info("피드백 감사합니다.")
```

---
## 실전 팁 (Best Practice)

- **데이터베이스/API 연동 시 필수:** Kafka나 DB에 쿼리를 날리는 작업은 무겁기 때문에 반드시 `st.form` 안에 넣어서 불필요한 호출을 막아야 합니다.

- **버튼 위치:** `form_submit_button`은 폼의 맨 마지막에 위치하는 것이 UX상 자연스럽습니다.