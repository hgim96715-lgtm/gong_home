---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Stream_Editor]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> **텍스트를 변환하는 문제입니다.**입력으로 들어오는 텍스트에서  
> **`thy`라는 단어가 나타나는 모든 부분을 `{}` 로 감싸서 출력**해야 합니다.

그리고 **대소문자를 구분하지 않습니다.**

1. **Input/Output 데이터 형태:**
    - **Input**: 각 줄에 **`thy`, `Thy`, `THY` 등 다양한 형태**로 등장할 수 있습니다.
    - **Output**: 모든 `thy`를 `{thy}` 형태로 감싸서 출력, 대소문자 구분 없이 처리
2. **핵심 명령어 & 로직 (Pipeline):**
	- `sed`로 `thy`를 찾음
3. **사용할 주요 옵션 (Flags):**
	- `g` 한 줄에서 모든 매치 치환
	- `I` 대소문자 구분 없이 검색
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

sed 's/thy/{&}/gI'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에 `thy`를 `{thy}`로 치환하려고 해서 `sed 's/thy/{thy}/gI'` 이 방법은 **매칭된 원래 문자열을 유지하지 못함**

-  **새로 알게 된 명령어/옵션:**
	- `&` : **sed에서 매칭된 원래 문자열을 의미**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
