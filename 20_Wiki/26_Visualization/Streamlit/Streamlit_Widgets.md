---
aliases:
  - Streamlit Widgets
  - 스트림릿 위젯
  - Input
  - Interactive
  - st.radio
  - 입력 위젯
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Layout]]"
  - "[[Streamlit_Status_Text]]"
linked:
  - file:///Users/gong/gong_study_de/streamlit/widget.py
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

#### **A. 기본 사용법**

```python
name = st.text_input("이름을 입력하세요:")
st.write("반갑습니다,", name)
```

#### **B. 실무 고급 파라미터 (`placeholder`, `key`, `type`)** 

단순한 입력창을 넘어, 사용자 경험(UX)을 올리고 버그를 막기 위해 실무에서는 아래 파라미터들을 적극적으로 활용합니다.

```python
search = st.text_input(
    label="🔍 노트 검색", 
    placeholder="파일명 검색...",    # 💡 꿀팁 1
    key=folder_path,               # 💡 꿀팁 2 (가장 중요!)
    type="password"                 # 💡 꿀팁 3 (비밀번호 가리기)
)
```

- **`key=folder_path` ( Streamlit의 핵심!):** 이 위젯의 **고유한 주민등록번호**를 부여합니다.
- **`type="password"` : 만약 이 옵션을 넣으면, 사용자가 타이핑하는 글자가 `****` 처럼 까맣게 가려집니다. 로그인 화면이나 API 키를 입력받을 때 필수
- **`placeholder="파일명 검색..."`**: '가이드 문구'입니다. 사용자가 클릭해서 타이핑을 시작하면 스르륵 사라집니다

### 슬라이더 (`st.slider`)

숫자(정수/실수)를 선택하거나 범위를 지정할 때 드래그해서 사용합니다.

- **단일 값:** 나이, 점수 등 하나의 값을 고를 때.
- **범위(Range):** `(25, 75)` 처럼 시작과 끝 값을 동시에 고를 때.

```python
# 0부터 100까지 중 선택 (기본값 25)
age = st.slider("나이 선택", 0, 100, 25)
```

### 버튼 (`st.button`)

클릭하면 특정 동작(Action)을 실행하는 트리거입니다.

- 주로 폼을 제출하거나 계산을 시작할 때 사용합니다.
- `if` 문과 함께 사용하여 "클릭되었을 때만" 코드가 실행되게 합니다.

```python
if st.button("Submit"):
	st.write(f"Hello {name}, you are {age}years old and love {language}!")
```

### 선택 박스 (`st.selectbox`)

사용자가 미리 정의된 리스트 중 단 하나를 고를 수 있게 해주는 **드롭다운(Dropdown) 메뉴**입니다. 
사용자의 입력을 엄격하게 통제하여 오타나 예외 값을 원천 차단합니다.

#### A. 기본 사용법 (`label`, `options`, `index`)

- `index` 파라미터로 앱이 켜졌을 때 처음 선택되어 있을 항목의 순번(0부터 시작)을 지정할 수 있습니다.

```python
option = st.selectbox(
    label="가장 좋아하는 언어는?",
    options=["Python", "Java", "JavaScript"],
    index=0 
)
```

#### B. 화면 노출값 변환 ⭐️ 실무 필수 (`format_func`)

내부적으로 컴퓨터가 쓰는 실제 값(DB Key 등)은 그대로 유지하면서, **화면에 그려지는 글씨(Display Label)만 예쁘게 포장**해 주는 강력한 기능입니다.

```python
CHALLENGES = {
    "ch_01": {"label": "파이썬 기초 챌린지", "difficulty": "Easy"},
    "ch_02": {"label": "데이터 분석 챌린지", "difficulty": "Hard"}
}

selected_key = st.selectbox(
    label="참여할 챌린지를 선택하세요",
    # options에는 'ch_01', 'ch_02' (Key 값)만 들어감
    options=list(CHALLENGES.keys()), 
    # 💡 핵심: 화면에 그릴 때는 딕셔너리의 "label" 값을 찾아와서 출력해라!
    format_func=lambda k: CHALLENGES[k]["label"]
)

# 🚨 선택된 값은 "파이썬 기초 챌린지"가 아니라 내부 Key인 "ch_01" 입니다!
st.write(f"내부적으로 선택된 데이터(Key): {selected_key}")
```

#### C. 미선택 상태 만들기 (`index=None`, `placeholder`)

처음에는 아무것도 선택되지 않은 빈칸(`None`) 상태를 만들고, 안내 문구를 띄웁니다.

```python
fruit = st.selectbox(
    label="과일을 고르세요",
    options=["사과", "바나나"],
    index=None,           # 처음엔 아무것도 선택 안 함
    placeholder="과일을 하나 골라주세요..." 
)
```

#### D. 인덱스(Index) 기반 선택 패턴 (`range` 활용)

`options`에는 리스트뿐만 아니라 `range()` 객체도 들어갈 수 있습니다. 
**문자열 자체가 아니라 리스트 내의 '위치 번호(인덱스)'를 결과값으로 반환받고 싶을 때** 아주 강력한 패턴입니다.

**원리:** `options`에 `0, 1, 2...` 같은 숫자를 넣고, 화면에 그릴 때만 `format_func`으로 리스트의 해당 인덱스 문자를 찾아와서 화장을 입히는 방식입니다.

```python
month_labels = ["1월", "2월", "3월", "4월"]

# 화면에는 "1월, 2월..."이 보이지만, 변수에 저장되는 값은 0, 1, 2, 3 이 됩니다.
selected_month_idx = st.selectbox(
    label="월 선택",
    # 💡 핵심: 0부터 3까지의 정수 범위를 옵션으로 제공
    options=range(len(month_labels)), 
    # 💡 화면에 그릴 때는 정수(i)를 받아서 리스트의 문자로 변환
    format_func=lambda i: month_labels[i]
)

# 선택된 값은 문자가 아니라 '숫자(인덱스)'이므로, 다른 리스트의 값을 찾을 때 유용합니다.
st.write(f"선택한 인덱스 번호: {selected_month_idx}")
st.write(f"실제 선택된 월: {month_labels[selected_month_idx]}")
```


---
## 심화 입력 위젯 & UI 최적화 (Advanced Inputs)

### 라디오 버튼 (`st.radio`) : 한눈에 보이는 단일 선택

`selectbox`와 비슷하지만, 선택지가 2~5개 정도로 적어서 **사용자가 옵션을 한눈에 파악해야 할 때** 직관적입니다.

**💡 실무 꿀팁: `format_func`와 `label_visibility` 활용하기** 딕셔너리의 '진짜 주소(Key)'는 숨기고 '예쁜 이름(Label)'만 화면에 띄우는 고급 기술입니다.

```python
CATEGORIES = {
    "20_Wiki/21": {"label": "Python"},
    "20_Wiki/22": {"label": "SQL"}
}

selected_key = st.radio(
	# `label="카테고리"`: 라디오 버튼 그룹의 제목(이름표)입니다.
    label="카테고리 선택", 
    options=list(CATEGORIES.keys()),
    #  핵심 치트키: 화면에는 예쁜 이름(label)만 띄워줍니다!
    format_func=lambda k: CATEGORIES[k]['label'], 
    #  UI 최적화: 제목 글자를 숨기고 여백까지 완전히 없애버립니다!("collapsed")
    # `"visible"` (기본값): 제목이 정상적으로 보입니다.
    # `"hidden"`: 제목 글자만 투명하게 숨기고, **그 글자가 차지하던 공간(여백)은 텅 빈 채로 남겨둡니다.** (줄 맞춤할 때 유용)
    label_visibility='collapsed',
    #  가로 배치 모드 , 기본적으로 세로
    horizontal=True 
)
# 사용자는 'Python'을 누르지만, 변수에는 '20_Wiki/21'이 저장됩니다.
```


### 체크박스 (`st.checkbox`)

ON/OFF 스위치처럼 동작합니다.

- **반환값:** 체크하면 `True`, 해제하면 `False` (Boolean).
- **용도:** 특정 차트를 숨기거나 보일 때, 옵션을 켤 때 사용합니다.

```python
show_table = st.checkbox("데이터 표 보기", value=True) # value=True는 기본 체크 상태

if show_table:
    st.write("여기 데이터 표가 보입니다!")
```


### 멀티 셀렉트 (`st.multiselect`)

`selectbox`와 비슷하지만, **여러 개를 동시에 선택**할 수 있습니다.

- **반환값:** 선택된 항목들이 담긴 **리스트(List)**.
- **용도:** 여러 지역 필터링, 태그 선택 등.

```python
regions = st.multiselect(
    "관심 지역을 선택하세요",
    ["서울", "경기", "인천", "부산"],
    ["서울", "경기"] # 기본 선택값 (Default)
)
st.write(f"선택한 지역: {regions}")
```

---
## 위치 및 꾸미기 (Layout & Text)

### 사이드바 (`st.sidebar`)

위젯을 화면 중앙이 아니라 **왼쪽 사이드바**에 고정하고 싶을 때 사용합니다.

- **사용법:** 모든 위젯 명령어 앞에 `st.sidebar.`만 붙이면 됩니다.
    
    - `st.button` → `st.sidebar.button`
    - `st.selectbox` → `st.sidebar.selectbox`

```python
# 왼쪽 사이드바에 연도 선택 박스 생성
year = st.sidebar.selectbox("연도 선택", [2023, 2024, 2025])
st.write(f"메인 화면: {year}년 데이터를 보여줍니다.")
```

### 텍스트 꾸미기 (`st.markdown`, `st.caption`,`st.html`,`st.divider`)

단순한 글자(`write`)보다 더 예쁘게 정보를 표시할 때 씁니다.

- **`st.markdown`:** 굵게, 기울임, 링크 등 마크다운 문법을 완벽 지원합니다. > `st.html` 도가능 [[Streamlit_Advanced_CSS#최신 CSS 주입 스킬과 실무 황금 공식 (`st.html` vs `st.markdown`)|st.html& st.markdown]] 참고 
- **`st.caption`:** 회색의 **작은 글씨**로 부연 설명을 달 때 사용합니다.
- **`st.divider()`:** 화면을 가로지르는 깔끔한 가로선(`---`)을 긋습니다.

```python
st.sidebar.markdown("---") # 구분선 긋기
st.sidebar.caption("Designed by Data Team") # 하단 저작권 표시 등에 활용
```


---



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

