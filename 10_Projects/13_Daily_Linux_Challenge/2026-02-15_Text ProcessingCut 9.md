---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Text_Slicing_cut]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 탭으로 구분된 여러 열로 구성된 파일(TSV 형식)이 주어졌을 때, 두 번째 열부터 마지막 ​​열까지 순서대로 출력하세요

1. **Input/Output 데이터 형태:**
    - **Input**: TSV 형식 파일
    - **Output**:  두 번째 열부터 마지막 ​​열까지 순서대로 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- cut은 -d를 안쓰면 기본은 탭 
3. **사용할 주요 옵션 (Flags):**
	- `cut`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

cut -f 2-
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
