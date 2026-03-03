---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Data_Statistics]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 주어진 텍스트 파일에서 연속해서 반복되는 줄을 제거하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 구성된 파일 내용
    - **Output**: 연속해서 중복된 줄이 제거된 텍스트
2. **핵심 명령어 & 로직 (Pipeline):**
	- `uniq` **인접한(연속된) 중복 줄만 제거**
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
uniq
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
