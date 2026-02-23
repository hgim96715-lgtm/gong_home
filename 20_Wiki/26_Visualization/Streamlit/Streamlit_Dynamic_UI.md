---
aliases:
  - Streamlit 동적 UI 생성
  - Python 문자열 렌더링
  - Dynamic UI
tags:
  - Streamlit
related:
  - "[[Streamlit_Advanced_CSS]]"
  - "[[Streamlit_Status_Text]]"
  - "[[Streamlit_Layout]]"
  - "[[Streamlit_Data_Display]]"
---
## 한줄 요약

> **"화면에 그릴 내용을 일일이 손으로 치지 마세요! 파이썬의 문자열 조작(`f-string`, `.join()`)을 이용해 UI 코드를 공장처럼 동적으로 생산해 내는 실무 1티어 패턴입니다."**

---
## 핵심 원리: "파이썬이 요리하고, Streamlit은 서빙만 한다"

많은 사람들이 착각하는 것 중 하나가 `st.html()`이나 `st.markdown()` 안에 파이썬 문법을 직접 쓴다고 생각하는 것입니다. 하지만 진실은 이렇습니다.

1. **파이썬(주방장):** `for`문과 `f-string`을 이용해 글자(String) 블록들을 열심히 조립합니다.
2. **완성된 문자열(요리):** `<span class='badge'>Kafka</span>` 또는 `## 환영합니다` 같은 완벽한 텍스트 덩어리가 완성됩니다.
3. **Streamlit(서빙 직원):** 완성된 텍스트 덩어리를 받아서 웹 브라우저 화면에 띡 하고 뿌려주기만 합니다.

---
## 실전 패턴 A: 동적 HTML 대량 생산 (`st.html`)

기술 스택 뱃지나 상태 태그처럼, 개수가 유동적인 요소를 그릴 때 하드코딩(복붙)을 방지하는 방법입니다.
**[[Streamlit_Advanced_CSS]]** 에서 만든 CSS와 함께 사용합니다.

### ❌ 하드코딩 (초보자의 방식)

기술이 추가될 때마다 HTML 태그를 직접 써야 해서 코드가 지저분해집니다.

```python
st.html("<span class='badge'>Kafka</span> <span class='badge'>Spark</span> <span class='badge'>Airflow</span>")
```

### ✅ 동적 렌더링 (엔지니어의 방식)

리스트에 데이터만 추가하면 화면은 알아서 그려집니다.

```python
import streamlit as st

# 1. 파이썬 리스트 (DB나 API에서 가져온 데이터라고 가정)
my_skills = ["Kafka", "Spark", "Airflow", "Docker", "AWS"]

# 2. 리스트 컴프리헨션 + f-string으로 HTML 태그 덩어리들 생성
# 3. .join()으로 덩어리들을 공백(" ")을 두고 하나로 합치기
html_string = " ".join([f"<span class='badge'>{skill}</span>" for skill in my_skills])

# 4. 완성된 거대한 HTML 문자열을 한 번에 렌더링
st.html(html_string)
```

---
## 실전 패턴 B: 동적 Markdown 대량 생산 (`st.markdown`)

마크다운의 제목(`##`), 굵은 글씨(`**`), 글머리 기호(`-`)도 똑같은 원리로 대량 생산할 수 있습니다. 
에러 로그나 작업(Task) 리스트를 띄울 때 강력합니다.

**f-string으로 변수 꽂아 넣기**

```python
user_name = "user"
today_processed = 150000

# f-string을 쓰면 마크다운 문법과 파이썬 변수를 섞어 쓸 수 있습니다.
st.markdown(f"### 👋 환영합니다, {user_name} 님!")
st.markdown(f"오늘 파이프라인에서 **{today_processed:,}건**의 데이터를 처리했습니다.")
```

**`.join()`으로 마크다운 리스트 생성하기**

```python
# 1. 실패한 작업 리스트
failed_tasks = ["Kafka_to_S3_Job", "Spark_Daily_Batch", "DB_Sync"]

# 2. 파이썬 문법으로 마크다운 텍스트 덩어리 만들기
markdown_list = "\n".join([f"- 🚨 **{task}** (실패)" for task in failed_tasks])

# 3. 완성된 텍스트를 마크다운으로 렌더링
st.markdown("### ❌ 오늘 실패한 배치 작업 목록")
st.markdown(markdown_list)
```

---
## 왜 이 패턴을 써야 할까? (실무의 이유)

**유지보수 극대화:** 
화면에 그릴 내용이 바뀌어도 HTML/Markdown 코드를 건드릴 필요 없이, 파이썬 리스트나 변수 값만 수정하면 됩니다.

**데이터 연동의 필수 관문:** 
DB(PostgreSQL, MySQL)에서 쿼리해 온 100줄의 결과값을 화면에 예쁘게 리스트로 뿌리려면 이 반복문 조립 기법이 무조건 필요합니다.
