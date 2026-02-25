---
aliases:
  - Streamlit_Mermaid
  - 스트림릿_머메이드
  - 다이어그램
  - 플로우차트
  - streamlit-mermaid
tags:
  - Streamlit
related:
  - "[[Streamlit_Charts]]"
  - "[[Python_Regex]]"
---

## 개념 한 줄 요약 (Concept Summary)

**"옵시디언에서 작성한 텍스트 기반 다이어그램(Mermaid)을 Streamlit 웹 화면에 진짜 예쁜 그림으로 렌더링해 주는 외부 컴포넌트 통합 기술."**

---

## 왜 필요한가 (Why is it needed)

 **문제점:** 
노션(Notion)이나 옵시디언(Obsidian)은 ` ```mermaid ` 블록을 쓰면 자동으로 예쁜 다이어그램을 그려줍니다. 
하지만 Streamlit의 기본 `st.markdown()`은 이를 지원하지 않고 그냥 **'단순한 코드 블록(글자)'** 으로만 뱉어냅니다.

 ** 해결책:** 
 `streamlit-mermaid` 라는 전용 서드파티 라이브러리를 설치하고, 마크다운 텍스트 안에서 Mermaid 코드만 쏙 빼내서 이 라이브러리에 던져주는 **'파싱(Parsing) & 렌더링'** 작업이 필요합니다.

---
## Practical Context (실무 활용)

* **아키텍처 문서화:** 데이터 파이프라인(Airflow DAG 흐름, Kafka-Spark 연동 구조 등)을 설명하는 포트폴리오 페이지를 만들 때, 이미지 파일을 일일이 캡처해서 올릴 필요 없이 마크다운 텍스트만으로 다이어그램을 렌더링할 수 있습니다.
* **유지보수 극대화:** 나중에 구조가 바뀌어도 포트폴리오 코드를 수정할 필요 없이, 옵시디언 노트의 글자 몇 개만 고치면 웹 화면의 다이어그램이 실시간으로 바뀝니다.

---
## Code Core Points: 완벽한 렌더링 로직 (폴백 패턴) 

텍스트와 다이어그램을 분리해서 렌더링하고, 라이브러리가 미설치되었을 때를 대비한 **현업 최고 수준의 방어적 프로그래밍(Fallback) 코드**입니다.

###  사전 준비 (설치)

터미널에서 라이브러리를 먼저 설치해야 합니다.

```bash
pip3 install streamlit-mermaid
```

### 마크다운 분해 및 렌더링 함수

마크다운 전체 텍스트를 받아서, 일반 글자와 다이어그램 코드를 분리한 뒤 번갈아 가며 화면에 그려주는 커스텀 함수입니다.

```python
import streamlit as st
import re

def render_md_with_mermaid(raw: str):
    """
    마크다운 텍스트 안에서 Mermaid 블록을 찾아 그림으로 렌더링하는 래퍼(Wrapper) 함수.
    """
    # 1. 정규식(Regex)을 이용해 머메이드 코드 블록 추출 (가상의 extract_mermaid 역할)
    # 실제로는 re.findall 등으로 ```mermaid ... ``` 안의 내용(diagrams)을 빼내고,
    # 그 자리를 '' 라는 임시 마커(Marker)로 치환합니다.
    diagrams, text = extract_mermaid(raw) 
    
    # 2. 불필요한 태그 정리 (가상의 clean_md 역할)
    cleaned = clean_md(text)
    
    # 3. 임시 마커를 기준으로 텍스트를 조각조각 자르기
    parts = cleaned.split('')
    
    # 4. 텍스트 조각과 다이어그램을 번갈아 가며 Streamlit에 그리기
    for i, part in enumerate(parts):
        # 4-1. 일반 마크다운 텍스트 출력
        if part.strip():
            st.markdown(part)
            
        # 4-2. 다이어그램 그리기 (마지막 텍스트 조각 뒤에는 다이어그램이 없을 수 있으므로 길이 체크)
        if i < len(diagrams):
            try:
                # 💡 핵심: 라이브러리를 동적으로 임포트 (가벼운 로딩 유지)
                from streamlit_mermaid import st_mermaid
                
                # 높이를 400px로 고정하여 예쁘게 렌더링
                st_mermaid(diagrams[i], height=400)
                
            except ImportError:
                # 🚨 방어적 프로그래밍 (Fallback)
                # 만약 클라우드 서버나 다른 환경에 라이브러리가 안 깔려있어서 에러가 나면,
                # 앱이 죽어버리게 두지 않고 그냥 '텍스트 코드' 형태로 보여준 뒤 안내 문구를 띄움!
                st.code(diagrams[i], language="text")
                st.caption("💡 `pip install streamlit-mermaid` 설치 시 다이어그램으로 예쁘게 표시됩니다!")
```

---
## Detailed Analysis

### 왜 `split('')` 를 쓸까?

-  Streamlit은 한 번에 화면 전체를 그리는 게 아니라, 블록(레고)을 하나씩 아래로 쌓는 구조입니다.
- 따라서 `마크다운 내용 ➡️ 다이어그램 그림 ➡️ 마크다운 내용` 순서대로 조립해주기 위해, 본문 텍스트를 마커를 기준으로 썰어서 배열(`parts`)로 만든 뒤 순서대로 `st.markdown()`과 `st_mermaid()`를 번갈아 호출하는 것입니다.