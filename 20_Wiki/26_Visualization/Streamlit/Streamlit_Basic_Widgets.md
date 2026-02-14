---
aliases:
  - Streamlit Widgets
  - 스트림릿 위젯
  - Input
  - Interactive
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"사용자와 앱이 대화하는 창구. 
HTML/JS 없이 파이썬 함수 하나로 텍스트 상자, 슬라이더, 버튼 같은 UI를 만든다."** 

---

## 왜 쓰는가? (Why)

단방향으로 정보만 보여주는 것이 아니라, **사용자의 입력(Input)** 을 받아서 앱을 **동적으로 제어(Control)** 하기 위해 사용합니다. 
* **No HTML/JS:** 복잡한 프론트엔드 코딩 없이 파이썬 함수만 부르면 즉시 인터랙티브 대시보드가 됩니다. 

---
## 핵심 위젯 4대장 (Core Widgets)

###  텍스트 입력 (`st.text_input`)

사용자가 문자열을 타이핑할 수 있는 입력창입니다. 
입력한 값은 바로 변수에 저장되어 코드에서 쓸 수 있습니다. 

```python
name = st.text_input("이름을 입력하세요:")
st.write("반갑습니다,", name)
```


### 슬라이더 (`st.slider`)

숫자(정수/실수)를 선택하거나 범위를 지정할 때 드래그해서 사용합니다.

- **단일 값:** 나이, 점수 등 하나의 값을 고를 때.
- **범위(Range):** `(25, 75)` 처럼 시작과 끝 값을 동시에 고를 때.

```python
# 0부터 100까지 중 선택 (기본값 25)
age = st.slider("나이 선택", 0, 100, 25)
```

### 선택 박스 (`st.selectbox`)

미리 정의된 리스트 중 하나를 고르는 드롭다운 메뉴입니다.

- 사용자의 입력을 특정 옵션으로 **제한(Restrict)** 하고 싶을 때 유용합니다.

```python
option = st.selectbox(
    "가장 좋아하는 언어는?",
    ["Python", "Java", "JavaScript"]
)
st.write("선택:", option)
```

### 버튼 (`st.button`)

클릭하면 특정 동작(Action)을 실행하는 트리거입니다.

- 주로 폼을 제출하거나 계산을 시작할 때 사용합니다.
- `if` 문과 함께 사용하여 "클릭되었을 때만" 코드가 실행되게 합니다.

```python
if st.button("Submit"):
	st.write(f"Hello {name}, you are {age}years old and love {language}!")
```

---
## 실전 활용 (Practical Context)

**데이터 엔지니어링 대시보드 예시:**

1. **`st.date_input`**: 조회하고 싶은 로그의 날짜 선택
2. **`st.selectbox`**: "결제 완료" vs "취소" 상태 필터링
3. **`st.button`**: "조회" 버튼을 누르면 그때 쿼리 실행 (비용 절약)

```python
# 간단한 조합 예시
category = st.selectbox("카테고리 선택", ["패션", "전자"])
if st.button("데이터 조회"):
    # 버튼을 눌렀을 때만 실행됨
    st.write(f"{category} 카테고리 데이터를 불러옵니다...")
```

