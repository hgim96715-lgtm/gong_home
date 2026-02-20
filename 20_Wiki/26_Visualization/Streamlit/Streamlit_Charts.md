---
aliases:
  - Streamlit Charts
  - 그래프
  - 시각화
  - Plotly
  - Matplotlib
  - Altair
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Concept]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"복잡한 설정 없이 `df`만 던져주면 바로 그려주는 '내장 차트'와, 디테일한 커스텀이 가능한 '외부 라이브러리' 연동 지원."**

---
##  Streamlit 내장 차트 (Built-in Charts)

별도의 라이브러리 설치 없이 가장 빠르게 데이터를 시각화할 때 사용합니다. 
Pandas DataFrame을 그대로 입력받습니다.

### ① 라인 차트 (`st.line_chart`)

* **용도:** 시계열 데이터(Time Series)나 추세(Trend)를 보여줄 때 가장 적합합니다.
* **특징:** 주식 차트나 센서 데이터처럼 시간이 지남에 따라 변하는 값을 그릴 때 씁니다.

### ② 바 차트 (`st.bar_chart`)

* **용도:** 카테고리별 데이터를 비교(Categorical Data Comparison)할 때 사용합니다.
* **특징:** 과일 판매량(사과 vs 바나나) 같은 이산형 데이터를 비교하기 좋습니다.

### ③ 지도 (`st.map`)

* **용도:** 위도(latitude)와 경도(longitude) 데이터를 지도 위에 점으로 찍어줍니다.
* **조건:** 데이터프레임에 반드시 `lat`, `lon` 이라는 이름의 컬럼이 있어야 자동으로 인식합니다.

---
## 외부 라이브러리 연동 (Advanced)

더 강력한 기능이나 특정 스타일이 필요할 때 사용합니다.

### ① Plotly (`st.plotly_chart`) - ⭐ 강력 추천

* **특징:** 기본적으로 **인터랙티브(Interactive)** 합니다.
* **기능:**
    * **Zoom:** 차트를 확대/축소할 수 있습니다.
    * **Hover:** 마우스를 올리면 상세 수치가 뜹니다.
    * **Pan:** 차트를 이리저리 움직일 수 있습니다.
* **활용:** 사용자가 직접 데이터를 탐색해야 하는 **대시보드**에 가장 적합합니다.

### ② Matplotlib (`st.pyplot`)

* **특징:** 정적(Static) 차트를 그릴 때 미세한 조정(Fine-grained control)이 가능합니다.
* **단점:** 그림(Image)처럼 고정되어 있어 줌이나 호버 기능이 없습니다. 논문용 그래프 스타일에 적합합니다.

### ③ Altair (`st.altair_chart`)

* **특징:** 선언형(Declarative) 통계 시각화 라이브러리입니다.
* **장점:** "어떻게 그릴지"가 아니라 **"무엇을 보여줄지"** 에 집중하는 문법을 가집니다. 기본적으로 인터랙티브 기능을 제공합니다.

---
## Code Example (Comparison)

```python
import streamlit as st
import pandas as pd
import numpy as np

# 샘플 데이터
df = pd.DataFrame(np.random.randn(20, 3), columns=['A', 'B', 'C'])

# 1. 내장 차트 (가장 간단)
st.subheader("1. Built-in Line Chart")
st.line_chart(df)

# 2. Plotly (인터랙티브)
import plotly.express as px
st.subheader("2. Plotly Chart")
fig = px.line(df, title="Interactive Plotly Line")
st.plotly_chart(fig)
```

---
##  자주 겪는 에러 
`
`There are multiple plotly_chart elements with the same auto-generated ID... please pass a unique key argument`

- **원인:**
	- Streamlit에서 실시간으로 숫자가 바뀌는 원리는 **"빈 의자(Placeholder)를 먼저 놓고, 나중에 사람(데이터)을 앉히는 방식"** 입니다.
	- 그런데 루프가 돌면서 예전 차트가 있던 똑같은 자리에 새로운 차트를 계속 밀어 넣으려니까 Streamlit이 헷갈려서 충돌이 난 것입니다.
- **해결 방안:** 
	- **"차트마다 고유한 이름표(`key`)를 달아줘!"** 차트를 그리는 함수 안에 `key` 파라미터를 추가하여 고유한 값을 넣어주면 해결됩니다.

```python
# 예시: 루프 카운터를 이용해 매번 새로운 이름표 달아주기
st.plotly_chart(fig, key=f"my_chart_{loop_count}")
```

> 빈의자(Placeholder) 전략 [[Streamlit_Concept]] 참고 