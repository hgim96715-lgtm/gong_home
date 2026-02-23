---
aliases:
  - Streamlit Install
  - 스트림릿 설치
  - 실행 방법
  - 환경변수 에러 해결
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
linked:
  - file:///Users/gong/gong_study_de/streamlit/requirements.txt
---

## 1. 설치하기 (Installation)

터미널에서 `pip3`를 사용하여 설치합니다. 특정 버전을 명시하는 것이 프로젝트 관리에 좋습니다.

### 방법 A: 한 줄로 설치 (간단)

```bash
pip install streamlit
# 또는 특정 버전
pip install streamlit==1.50.0
```

### 방법 B: requirements.txt 사용 (권장, 이방법으로 설치했다)

협업이나 배포를 위해 파일로 관리하는 정석 방법입니다.

#### `requirements.txt` 파일 생성

```text
# === Web Application Framework ===
streamlit==1.50.0   # 웹 앱을 만들어주는 주인공

# === Visualization Tools (시각화) ===
matplotlib==3.9.4   # [기본] 정적인 차트 (움직이지 않는 이미지) 그릴 때 사용
plotly==6.3.1       # [고급] 동적인 차트 (줌, 호버 가능한 인터랙티브) 그릴 때 사용

# === Data Processing (데이터 처리) ===
pandas              # 엑셀처럼 표(DataFrame) 데이터를 다루는 필수 라이브러리
numpy               # 숫자 계산, 행렬 연산 등을 빠르게 처리하는 도구

# === Backend & System (백엔드 및 시스템) ===
kafka-python        # (Kafka 연동 시) Kafka 서버와 데이터를 주고받기 위해 필수
watchdog            # (선택) 코드를 수정하고 저장하면, 웹 화면을 자동으로 새로고침 해줌 (개발 편의성 Up!)
```

**💡 팁:** `kafka-python`이 없으면 Streamlit에서 Kafka 데이터를 못 읽어옵니다. 꼭 추가!!

#### 설치 명령어 실행:

```bash
pip3 install -r requirements.txt
```

---
## 파일 생성 (Hello World)

실행 테스트를 위해 프로젝트 폴더에 `Home.py`를 만듭니다.

```python
import streamlit as st

st.title("🎉 설치 성공!")
st.write("나의 첫 번째 Streamlit 앱입니다.")
st.balloons()  # 풍선 효과import streamlit as st
```

---
## 실행하기 (Execution) - 중요!

설치 직후 `command not found` 에러가 뜬다면 당황하지 말고 **방법 B**를 쓰세요.

### 방법 A: 표준 명령어 (Standard)

환경 변수(PATH)가 잘 잡혀 있을 때 사용합니다.

```bash
streamlit run Home.py
```

> 가상방법 (venu)활용

```bash
# 1. 가상환경 생성 (최초 1회)
python3 -m venv venv

# 2. 가상환경 활성화 (프로젝트 시작할 때마다)
source venv/bin/activate

# 3. 이제 그냥 사용 가능!
pip3 install streamlit
streamlit run portfolio.py
```


### 방법 B: 절대 실패하지 않는 명령어 (Robust)

`streamlit` 명령어를 못 찾을 때(`zsh: command not found`), 파이썬 모듈(`-m`)로 직접 호출하는 확실한 방법입니다.

```bash
python3 -m streamlit run Home.py
```

**왜 이 방법을 쓰나요?** 
맥북이나 가상환경 설정에 따라 터미널이 `streamlit` 실행 파일의 위치를 모를 수 있습니다.
이때 `python3 -m`을 붙이면 **"파이썬아, 네가 설치한 streamlit 모듈 좀 실행해줘!"** 라고 명확하게 지시하는 것이라 에러가 나지 않습니다.

---
## 확인하기 (Verification)

1. 명령어를 입력하면 터미널에 `Local URL: http://localhost:8501`이 뜹니다.
2. 브라우저가 자동으로 열리며 웹사이트가 보입니다.
3. 화면에 풍선(`balloons`)이 날아오르면 성공입니다

---
## 종료하기 (Stop)

실행 중인 터미널에서 **`Ctrl + C`** 를 누르면 서버가 종료됩니다.


