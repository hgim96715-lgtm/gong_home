---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[File_Content_Viewing]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 입력 파일의 끝부분에 있는 문자들을 출력

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트파일
    - **Output**: 해당 파일의 **마지막 20바이트(byte)에 해당하는 문자들**
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `tail`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

tail -c 20
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
