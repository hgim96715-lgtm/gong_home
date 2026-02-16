---
aliases:
  - st.metric
  - kpi
  - scoreboard
  - dashboard_stats
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Concept]]"
---
## 개념 한 줄 요약

**"주식 앱이나 스포츠 스코어보드처럼, 중요한 숫자 하나를 강조해서 보여주는 전용 컴포넌트."**

---
## 기본 구조 (`st.metric`)

대시보드 맨 위에 배치되는 **핵심 성과 지표(KPI)** 를 보여줄 때 사용합니다.
글자 크기를 키우거나 색칠할 필요 없이, 함수 하나로 깔끔한 디자인이 완성됩니다.

```python
import streamlit as st

# 기본 사용법
st.metric(label="현재 온도", value="24 °C")
```

- **label**: 회색 작은 글씨 (제목)
- **value**: 검은색 큰 글씨 (값)

---
## 변화량 보여주기 (`delta`) 

`metric`의 진정한 강력함은 **"어제보다 올랐나/내렸나"** 를 보여주는 화살표 기능에 있습니다.

```python
col1, col2, col3 = st.columns(3)

with col1:
    # 🟢 초록색 화살표 (상승)
    st.metric(label="삼성전자", value="70,000 원", delta="1,200 원")

with col2:
    # 🔴 빨간색 화살표 (하락)
    st.metric(label="SK하이닉스", value="120,000 원", delta="-500 원")

with col3:
    # ⚪️ 회색 (변화 없음)
    st.metric(label="보합", value="500", delta="0")
```

### 💡 색상 반전시키기 (`delta_color`)

주식과 달리, **"에러 횟수"** 나 **"대기 시간"** 은 줄어들수록 좋은 거죠? 이럴 때는 색상 로직을 뒤집을 수 있습니다.

```python
# 값이 줄어들었는데(-5ms) 초록색(Good)으로 표시됨!
st.metric(
    label="응답 속도", 
    value="120ms", 
    delta="-5ms", 
    delta_color="inverse"  # (기본값은 'normal')
)
```

---
## 실전 배치 (Columns와 찰떡궁합)

`st.metric`은 혼자 쓰기보다 **`st.columns`** 와 함께 상단에 가로로 나열할 때 가장 예쁩니다.

```python
# 대시보드 헤더 영역
m1, m2, m3, m4 = st.columns(4)

m1.metric("총 매출", "$5,200", "+5%")
m2.metric("방문자 수", "300명", "-20명")
m3.metric("가입자 수", "12명", "+1")
m4.metric("서버 상태", "OK")
```

---
## 실시간 업데이트 (빈 의자 전략)

Kafka 로그처럼 계속 바뀌는 숫자를 보여줄 때는 `st.empty()`와 함께 씁니다.

```python
import time

# 1. 자리 잡기
kpi_placeholder = st.empty()

# 2. 계속 덮어쓰기
for score in range(100):
    kpi_placeholder.metric(
        label="실시간 점수", 
        value=score, 
        delta=f"+{score * 0.1}"
    )
    time.sleep(0.1)
```

