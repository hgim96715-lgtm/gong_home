---
aliases:
  - st.dataframe
  - use_container_width
  - st.empty
  - 데이터프레임출력
tags:
  - Pandas
  - Streamlit
related:
  - "[[Pandas_DataStructures]]"
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Concept]]"
---
## 개념 한 줄 요약

**"Pandas로 가공한 데이터프레임(표)을 웹 브라우저 화면에 '인터랙티브'하게 뿌려주는 스트림릿 전용 함수"**

---
## 기본 문법 (`st.dataframe`)

`print(df)`는 터미널에 텍스트로 찍히지만, `st.dataframe(df)`는 **정렬, 검색, 스크롤**이 가능한 엑셀 같은 표를 웹에 그려줍니다.

```python
import streamlit as st
import pandas as pd

df = pd.DataFrame({'A': [1, 2, 3], 'B': [4, 5, 6]})

# 웹 화면에 표 출력
st.dataframe(df)
```

---
## 핵심 파라미터 분석 (`use_container_width`) 

```python
category_table_slot.dataframe(df_summary, use_container_width=True)
```

- **`data` (첫 번째 인자):** 출력할 Pandas DataFrame (`df_summary`).
- `use_container_width=True`
	- **True:** 표를 화면(또는 컬럼) 너비에 **꽉 차게** 늘립니다. (반응형 웹처럼 시원시원해 보임)
	- **False (기본값):** 데이터 내용만큼만 작게 표시됩니다.
- **`hide_index=True`:** 행 번호(0, 1, 2...)가 보기 싫을 때 숨기는 옵션입니다. (자주 씀!)

---
## 빈칸 채우기 (`st.empty`와 `slot`)

질문하신 코드 앞에 있는 `category_table_slot`의 정체입니다.

- **`st.empty()`**: 화면에 미리 **"자리(Slot)"** 만 찜해두는 기능입니다.
- 나중에 데이터가 들어오면 그 자리를 덮어씁니다. (실시간 대시보드 필수템!)

```python
# 1. 화면 구석에 빈 자리를 하나 만듭니다. (아직은 아무것도 안 보임)
my_slot = st.empty()

# ... (데이터 처리 중) ...

# 2. 아까 찜해둔 자리에 표를 그립니다.
# 그냥 st.dataframe()을 쓰면 순서대로 밑에 계속 쌓이지만,
# slot.dataframe()을 쓰면 이 자리 내용만 계속 바뀝니다. (깜빡임 방지)
my_slot.dataframe(df, use_container_width=True)
```

---
## `st.table`과의 차이점

- **`st.dataframe`**: **동적**. 스크롤 가능, 크기 조절 가능, 정렬 가능. (데이터 많을 때 추천)
- **`st.table`**: **정적**. 표 전체를 그냥 쫘악 펼쳐서 보여줌. (데이터가 적을 때 추천)