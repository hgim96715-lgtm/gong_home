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

> 마지막 값을 출력하는 방법을 알아봅니다.텍스트 파일의 줄들.
> 마지막을 표시합니다입력 파일의 줄.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트 파일.
    - **Output**: 마지막 20줄 
2. **핵심 명령어 & 로직 (Pipeline):**
	- `tail`

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
tail -n 20
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
