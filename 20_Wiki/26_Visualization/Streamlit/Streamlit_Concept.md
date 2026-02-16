---
aliases:
  - Streamlit
  - 스트림릿
  - Vibe Coding
  - Data App
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
  - "[[Streamlit_Data_Display]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"HTML/CSS 없이 파이썬 스크립트만으로 데이터 사이언스용 웹 앱을 즉시 만들어주는 오픈소스 프레임워크."** 

---

## 왜 쓰는가? (Why: Vibe Coding)

기존 웹 개발은 **"코드 수정 → 빌드 → 배포 → 확인"** 의 긴 사이클이 필요했습니다. 
하지만 Streamlit은 **"Vibe Coding(바이브 코딩)"**을 가능하게 합니다.

1.  **Zero Front-end Skills:** HTML, CSS, JavaScript, React를 몰라도 됩니다. 파이썬만 할 줄 알면 웹 앱을 만들 수 있습니다.
2.  **Immediate Feedback (즉각 피드백):** 파이썬 코드를 저장하는 순간 브라우저가 새로고침되어 결과가 바로 뜹니다.
    * *Vibe Coding Cycle:* "시도하고 → 결과를 보고 → 수정하고 → 반복".

---
## 핵심 특징 (Key Features)

### ① Beautiful by Default (알아서 예쁨)

디자인 감각이 없어도 됩니다. 
버튼, 슬라이더, 차트, 텍스트 등 모든 UI가 **기본적으로 깔끔하고 현대적(Clean and Modern)** 으로 나옵니다.
초보자가 만들어도 전문가가 만든 것처럼 보입니다.

### ② Built-in Data & ML Support

데이터 엔지니어링/사이언스에 최적화되어 있습니다.
* **호환성:** Pandas, NumPy, Matplotlib, Plotly 등과 완벽하게 연동됩니다.
* **시각화:** 복잡한 데이터나 ML 예측 결과를 표나 그래프로 즉시 보여줍니다.

### ③ Easy Sharing (쉬운 공유)

콘솔창(검은 화면)에 결과만 띄우는 건 이제 그만! 
`streamlit run app.py` 명령어 하나면 남들에게 보여줄 수 있는 웹 사이트가 됩니다. 
"말보다는 보여주는 것(Show, don't tell)"이 중요한 포트폴리오나 해커톤에 최적입니다.

---
## 핵심 동작 원리: 빈 의자 전략 (Placeholder)

Streamlit은 기본적으로 코드를 위에서 아래로 실행하며 화면을 그립니다. (`append` 방식)
하지만 실시간 대시보드(Kafka 로그, 타이머)를 만들 때는 **데이터가 밑으로 쌓이는 게 아니라, 제자리에서 숫자만 바뀌어야 합니다.**

이때 **"빈 의자(Placeholder)"** 개념을 사용합니다.

1. **공간 확보 (`st.empty`):** 먼저 빈 상자(Container)를 화면에 배치하여 자리를 찜합니다.
2. **변신 (`.method`):** 그 상자에게 명령을 내려서 내용을 채웁니다.
3. **교체:** 새로운 명령을 내리면 기존 내용은 사라지고 새 내용으로 **덮어씌워집니다.**

>`st.empty()`로 만든 객체는 단순 글자(`write`)뿐만 아니라 **상태 메시지, 차트, 데이터프레임** 등 무엇이든 될 수 있습니다.


```python
import time
import streamlit as st

# 1. 빈 의자(공간) 먼저 놓기
my_slot = st.empty()

# 2. 상황에 따라 변신시키기
my_slot.info("⏳ 데이터 수신 대기 중...")  # 파란 알림창
time.sleep(2)

my_slot.success("✅ Kafka 연결 성공!")    # 초록 성공창 (기존 파란창은 사라짐)
time.sleep(2)

my_slot.error("🚫 네트워크 오류 발생!")     # 빨간 에러창 (기존 초록창은 사라짐)
```

#### **💡 만약 `st.empty()`를 안 쓰면?**

숫자가 `0, 1, 2...`로 제자리에서 바뀌는 게 아니라, 화면 아래로 `0` `1` `2` ... 하고 무한히 스크롤이 생기며 쌓이게 됩니다.

`st.empty()`는 "뭐든지 담을 수 있는 마법 상자"입니다. 
`.write()`뿐만 아니라 `.success()`, `.line_chart()`, `.dataframe()` 등 `st.` 뒤에 붙는 거의 모든 명령어를 사용할 수 있습니다.

---

## 실전 맥락 (Practical Context)

**언제 쓰는가?**
- **데이터 엔지니어:** Kafka로 들어오는 실시간 로그를 모니터링하는 대시보드를 만들 때.
- **머신러닝 엔지니어:** 모델의 예측 결과를 비개발자(기획자, 마케터)에게 시각적으로 보여줄 때.
- **취준생/학생:** 복잡한 배포 과정 없이 내 프로젝트를 근사한 포트폴리오로 만들 때.

---

## Code Core Points

**기존 방식 vs Streamlit 방식**

```python
# [기존] 콘솔 출력 (나만 봄)
print("Hello World")
print(df)

# [Streamlit] 웹 앱 출력 (남들에게 공유 가능)
import streamlit as st

st.title("Hello World")  # 제목으로 예쁘게 출력
st.write(df)             # 데이터프레임을 인터랙티브 표로 변환
```
