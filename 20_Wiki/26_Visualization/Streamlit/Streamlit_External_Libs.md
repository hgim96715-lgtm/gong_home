---
aliases:
  - Streamlit 외부 라이브러리
  - requests
  - base64
  - re
  - 정규표현식
tags:
  - Python
  - Streamlit
related:
  - "[[Python_Regex]]"
  - "[[Python_Requests_Response]]"
  - "[[Streamlit_Dynamic_UI]]"
  - "[[00_Streamlit_HomePage]]"
---
# Streamlit의 한계를 부수는 3대장

>"Streamlit 기본 기능만으로는 2% 부족할 때, 화면을 더 예쁘게 꾸미거나 외부 데이터를 끌어오는 실무 치트키 라이브러리 3총사입니다."

---
## `import requests` (파이썬의 웹 브라우저)

**정체:** 인터넷 세상과 통신(HTTP)을 주고받게 해주는 국민 라이브러리입니다.

- **Streamlit에서 언제 쓰나요?**
    - **외부 API 데이터 가져올 때:** 날씨 데이터, 주식 데이터, 환율 등을 URL로 찔러서 실시간으로 가져올 때 씁니다.
    - **Lottie 애니메이션 넣을 때:** 움직이는 예쁜 일러스트(Lottie JSON 파일)를 웹에서 다운로드해서 화면에 띄울 때 무조건 `requests.get(url)`을 사용합니다.

```python
import streamlit as st
import requests

# 실무 예시: Lottie 애니메이션 URL에서 가져오기
url = "https://assets.lottiefiles.com/..."
response = requests.get(url)
data = response.json() # 화면에 띄울 준비 완료!
```

---
## `import base64` (데이터를 안전하게 포장하고 해독하는 마술사)

"HTML에 내 사진을 직접 넣을 수 없을 때 사진을 알파벳 텍스트로 바꾸거나, 외부 API(GitHub 등)가 안전하게 암호화해서 보낸 문서의 포장을 뜯어낼 때 사용합니다!"

**정체:** 이미지나 파일, 혹은 깨지기 쉬운 특수문자가 섞인 원본 데이터를 인터넷(JSON, HTML 등)에서 안전하게 주고받기 위해 **'인간이 읽을 수 있는 기본 알파벳(ASCII) 텍스트'**로 변환(Encoding)하거나, 원래대로 해독(Decoding)해 주는 모듈입니다.

Streamlit에서 언제 쓰나요? (프론트엔드 & 백엔드 필수 스킬!)


### ① 프론트엔드 UI 꾸밀 때 (Encoding : 로컬 이미지를 HTML로)

`st.html()`이나 `st.markdown()`으로 커스텀 디자인을 할 때, 로컬 이미지(내 컴퓨터에 있는 프로필 사진 등)를 CSS 백그라운드로 넣는 것은 매우 까다롭습니다. 
이때 내 사진을 `base64`로 변환해서 아주 긴 텍스트로 만든 다음, HTML `<img src="...">` 태그 안에 텍스트 형태로 직접 꽂아 넣습니다.

```python
import base64

# 실무 예시: 로컬 이미지를 base64 텍스트로 변환하기
with open("profile.png", "rb") as image_file:
    encoded_string = base64.b64encode(image_file.read()).decode()

# 변환된 텍스트를 HTML에 직접 주입!
st.html(f'<img src="data:image/png;base64,{encoded_string}">')
```

### ② 백엔드 API 연동할 때 (Decoding : GitHub 데이터 해독하기)

GitHub 같은 외부 API는 마크다운(.md) 파일 내부의 줄바꿈이나 특수문자가 통신 중 깨지는 것을 막기 위해, 파일 내용 전체를 `base64`로 암호화해서 보내줍니다. 이걸 Streamlit 화면에 뿌려주려면 다시 해독(디코딩)해야 합니다.

```python
import base64
import requests

# 실무 예시 2: GitHub API가 base64로 꽁꽁 싸매서 보낸 마크다운 문서 해독하기
def get_md_content(path: str) -> str:
    url = f"https://api.github.com/repos/{GITHUB_USER}/{GITHUB_REPO}/contents/{path}?ref={BRANCH}"
    r = requests.get(url, timeout=10)
    
    if r.status_code == 200:
        data = r.json()
        
        # 1단계: b64decode()로 base64 포장지 뜯기 (바이트 데이터로 변환됨)
        # 2단계: decode("utf-8")을 통해 우리가 아는 한글/영문 텍스트로 최종 변환!
        content = base64.b64decode(data['content']).decode("utf-8")
        
        return content
    return "문서를 불러오지 못했습니다."
```

---
## `import re` (슈퍼마이크로 '찾기 및 바꾸기')

**정체:** 정규표현식(Regular Expression)의 약자입니다. 텍스트 안에서 특정한 '패턴'을 찾아내는 초강력 돋보기입니다.

**Streamlit에서 언제 쓰나요?**

- **유효성 검사:** 사용자가 화면의 텍스트 박스(`st.text_input`)에 전화번호나 이메일을 입력했을 때, 형식에 맞게 잘 입력했는지 검사할 때 씁니다. (예: `^[a-zA-Z0-9]+@[a-zA-Z0-9]+\.[a-z]+$` 이메일 패턴 검사)
- **데이터 정제(Cleansing):** 웹에서 크롤링한 지저분한 데이터나 유저가 입력한 텍스트에서 숫자나 한글만 깔끔하게 정제해서 화면에 뿌려줄 때 씁니다.

```python
import re
import streamlit as st

email = st.text_input("이메일을 입력하세요")

# 실무 예시: 정규표현식으로 이메일 형식 검사하기
if email:
    pattern = r'^[a-zA-Z0-9]+@[a-zA-Z0-9]+\.[a-z]+$'
    if re.match(pattern, email):
        st.success("올바른 이메일 형식입니다!")
    else:
        st.error("이메일 형식에 맞게 적어주세요.")
```

