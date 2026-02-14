---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[Text_Slicing_cut]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 주어진 문장의 처음 세 단어를 찾아 표시하세요. 단어 사이에는 공백(' ')만 있다고 가정합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트 파일
    - **Output**: 각 입력 문장에서 처음 세 단어
2. **핵심 명령어 & 로직 (Pipeline):**
	- `cut`
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
cut -d " " -f 1-3
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
