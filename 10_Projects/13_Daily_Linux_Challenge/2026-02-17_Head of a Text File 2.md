---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[File_Content_Viewing]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> **텍스트 파일의 처음 20글자(characters)** 를 출력하는 연습을 합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트파일 
    - **Output**:  텍스트 파일에서 **앞에서부터 20글자만 출력**
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `head`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

head -c 20
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
