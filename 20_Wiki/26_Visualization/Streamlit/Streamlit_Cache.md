---
aliases:
  - Cache
  - 캐싱
  - 성능 최적화
  - st.cache_data
  - st.cache_resource
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Concept]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"오래 걸리는 연산 결과를 메모리에 저장해두고, 똑같은 요청이 오면 계산 없이 저장된 결과를 바로 꺼내주는 기술."** 

---
## 왜 써야 하나요? (The Problem)

Streamlit은 버튼을 누르거나 위젯을 건드릴 때마다 **코드 전체를 처음부터 끝까지 다시 실행(Rerun)** 하는 특징이 있습니다.
* **문제점:** 1GB짜리 데이터를 불러오는데 10초가 걸린다면, 버튼을 누를 때마다 사용자는 10초씩 기다려야 합니다.
* **해결책:** "데이터 로딩" 함수에 캐시를 걸어두면, 첫 번째 실행 때만 10초가 걸리고 두 번째부터는 0.1초 만에 로딩됩니다.

---
##  캐시의 두 가지 종류 (Two Types)

Streamlit은 용도에 따라 두 가지 데코레이터(Decorator)를 제공합니다. 
상황에 맞춰 골라 써야 합니다.

| 데코레이터 | 용도 (Purpose) | 사용 시점 (When to Use) | 특징 |
| :--- | :--- | :--- | :--- |
| **`@st.cache_data`** | **데이터 저장** | 엑셀/CSV 파일 로딩, 데이터프레임 연산, API 호출 결과 | 데이터를 복사(Copy)해서 반환함 (안전함)  |
| **`@st.cache_resource`** | **자원 공유** | ML 모델 로딩, 데이터베이스(DB) 연결 | 자원 자체를 참조(Reference)함 (빠름, 공유됨)  |

---
## 사용 방법 (How to Use)

함수 정의 바로 위에 `@데코레이터`만 붙여주면 끝입니다.

### ① 데이터 로딩 최적화 (`st.cache_data`)

가장 많이 쓰이는 패턴입니다. 무거운 CSV 파일을 읽을 때 사용합니다.

```python
import streamlit as st
import pandas as pd
import time

# 이 함수는 캐싱되므로, 인자(filename)가 같다면 재실행되지 않음
@st.cache_data  # <--- 마법의 명령어!
def load_big_data(filename):
    time.sleep(3) # 3초 걸리는 작업을 가정
    df = pd.read_csv(filename)
    return df

st.write("데이터 로딩 시작...")
df = load_big_data("data.csv") # 처음엔 3초, 그다음부턴 0초!
st.write("로딩 완료!", df)
```


### ② AI 모델/DB 연결 최적화 (`st.cache_resource`)

텐서플로우 모델이나 DB 커넥션처럼, 계속 연결을 유지해야 하는 무거운 객체에 사용합니다.

```python
@st.cache_resource
def load_ai_model():
    # 모델 로딩은 아주 무거운 작업
    model = "Super_AI_Model_Loaded" 
    return model

model = load_ai_model() # 앱이 리런돼도 모델을 다시 로드하지 않음
```

----
## 캐시 초기화 (Invalidation)

데이터가 업데이트되었거나 강제로 새로고침을 해야 할 때 사용합니다

```python
if st.button("최신 데이터로 새로고침"):
    st.cache_data.clear()      # 데이터 캐시 비우기 
    st.cache_resource.clear()  # 리소스 캐시 비우기
```

---
## 실전 팁 (Best Practices)

1. **함수 단위로 쪼개기:** 캐싱은 "함수" 단위로 작동하므로, 무거운 작업은 반드시 별도 함수로 만들어서 데코레이터를 붙이세요.
2. **매개변수(Argument) 활용:** 함수에 들어가는 인자 값이 바뀌면, Streamlit은 "새로운 요청"으로 인식하고 코드를 다시 실행합니다.