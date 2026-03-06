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

> 텍스트 파일이 주어졌을 때 **앞줄이나 뒷줄과 동일하게 반복되지 않는 줄만 출력**하는 문제이다.  
> 즉 **연속해서 같은 문자열이 나타나는 경우는 제거하고**, **혼자 등장하는 줄만 출력**한다.  
> 문자열 비교는 **대소문자를 구분(case-sensitive)** 한다.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 이루어진 텍스트 파일. 각 줄에는 문자열이 하나씩 들어 있으며 같은 문자열이 **연속해서 여러 번 등장할 수 있음**.
    - **Output**: **앞줄이나 뒷줄과 동일한 문자열이 없는 줄(연속 중복이 아닌 줄)** 만 그대로 출력.
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `unique` — 고유값만 출력 `-u`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

uniq -u
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
