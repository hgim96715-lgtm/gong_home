---
aliases:
  - select_dtypes
  - 데이터 타입 필터링
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Pandas_Selection]]"
---
## 개념 한 줄 요약

**"수많은 컬럼 중 숫자만, 혹은 문자열만 있는 컬럼들만 광속으로 골라내는 기능"**

---
## 왜 쓰는가? (실전 사례)

데이터를 분석하다 보면 컬럼이 100개가 넘을 때가 있습니다. 
- **시각화:** "차트를 그려야 하는데, 숫자 타입 컬럼이 뭐뭐 있지?"
- **전처리:** "문자열(Object) 컬럼들만 골라서 빈칸을 채워야지."
- **머신러닝:** "모델 학습에 쓸 숫자 데이터만 따로 뽑아야겠어."

이때 하나하나 컬럼 이름을 적지 않고 **타입(Type)** 으로 한 방에 해결합니다.

---
## 기본 문법 (`select_dtypes`)

```python
import pandas as pd

# 1. 숫자 타입만 가져오기 (int, float 포함)
numeric_df = df.select_dtypes(include=['number'])

# 2. 문자열(Object) 타입만 가져오기
string_df = df.select_dtypes(include=['object'])

# 3. 날짜 타입만 가져오기
date_df = df.select_dtypes(include=['datetime'])
```

---
## 반대로 제외하기 (`exclude`)

"이 타입만 빼고 다 가져와!"라고 할 수도 있습니다.

```python
# 문자열(object) 컬럼만 빼고 전부 다 가져오기
no_string_df = df.select_dtypes(exclude=['object'])
```

---
## 실전 활용 (Streamlit 대시보드 예시)

Kafka에서 넘어온 로그 데이터 중 **숫자 데이터**만 찾아서 실시간 선 그래프를 그릴 때 사용합니다.

```python
# 1. 데이터프레임에서 숫자 컬럼 이름들만 리스트로 뽑기
numeric_cols = df.select_dtypes(include=['number']).columns

# 2. 만약 숫자 컬럼이 하나라도 있다면 차트 그리기
if len(numeric_cols) > 0:
    st.line_chart(df[numeric_cols])
```

**팁:** > `.columns`를 뒤에 붙이면 데이터프레임이 아니라 **컬럼 이름(리스트 형태)** 만 깔끔하게 얻을 수 있습니다.