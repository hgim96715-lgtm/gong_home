---
aliases:
  - Streamlit Secrets
  - secrets.toml
  - 환경변수
  - API Key 관리
  - 보안 설정
tags:
  - Streamlit
related:
  - "[[00_Streamlit_HomePage]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"API 키, 비밀번호 같은 민감한 정보를 코드에서 분리하여 `.streamlit/secrets.toml` 파일에 따로 저장하고 관리하는 기능."** 

---
## 왜 써야 하나요? (Why)

초보자가 가장 많이 하는 실수가 파이썬 코드 안에 비밀번호를 직접 적는 것(Hardcoding)입니다. 

* **위험성:** 코드를 Git에 올리는 순간 전 세계 해커들이 내 API 키를 훔쳐갈 수 있습니다.
* **해결책:** `secrets.toml` 파일에만 정보를 적고, 이 파일은 Git에 올리지 않도록 설정하여 보안을 지킵니다. 

---
##  파일 생성 및 위치 (File Location)

프로젝트 최상위 폴더에 `.streamlit`이라는 숨김 폴더를 만들고, 그 안에 파일을 생성해야 합니다. 

* **경로:** `.streamlit/secrets.toml`
* **주의:** 파일명(`secrets`)과 확장자(`toml`)가 정확해야 합니다.

---
## 작성 문법 (Syntax: TOML)

`toml` 파일은 윈도우의 `ini` 파일과 비슷합니다. 대괄호 `[]`로 섹션을 나누고, `키 = "값"` 형태로 저장합니다. 

**작성 예시 (`.streamlit/secrets.toml`)**

```toml
# 1. 일반 설정
[general]
app_name = "My Streamlit App"

# 2. 데이터베이스 접속 정보
[database]
user = "admin"
password = "super_secret_password!@#"
host = "localhost"
dbname = "mydb"

# 3. API 키 관리 (OpenAI, AWS 등)
[api_keys]
openai = "sk-1234567890abcdef..."
aws = "AKIA..."
```

---
## 코드에서 불러오기 (How to Access)

코드에서는 `st.secrets` 명령어를 사용하여 딕셔너리처럼 값을 꺼내 씁니다

```python
import streamlit as st

# 1. 간단한 접근
st.write(f"앱 이름: {st.secrets['general']['app_name']}")

# 2. API 키 가져오기
api_key = st.secrets["api_keys"]["openai"]

# 3. DB 접속 (활용 예시)
db_user = st.secrets["database"]["user"]
db_pw = st.secrets["database"]["password"]

st.write(f"DB 유저 '{db_user}'로 접속을 시도합니다...")
```

**특징:** `st.secrets`는 **읽기 전용(Read-only)** 딕셔너리처럼 작동합니다. 코드에서 값을 수정할 수 없습니다.

---
## ⚠️ 필수 보안 수칙 (Best Practices)

이 파일(`secrets.toml`)을 만들었다면 반드시 **`.gitignore`** 파일에 추가해야 합니다

```.gitignore
# Streamlit secrets 
.streamlit/secrets.toml
```

이렇게 해야 Git이 이 파일을 무시하고, 깃허브에 비밀번호가 올라가는 대참사를 막을 수 있습니다.

---
## 활용 사례 (Use Cases)

- **DB 로그인 정보:** `host`, `port`, `user`, `password` 등
- **API 토큰:** OpenAI, Hugging Face, Google Maps API 키
- **데모용 계정:** 프로토타입 앱의 임시 관리자 ID/PW

