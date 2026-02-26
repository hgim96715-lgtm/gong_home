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

> 각 줄에 숫자가 포함된 텍스트 파일이 주어집니다. 숫자는 정수이거나 소수점이 있을 수 있습니다. 각 줄 끝에는 숫자와 줄 바꿈 문자 외에 다른 문자가 없습니다. 가장 작은 숫자가 첫 번째 줄에, 가장 큰 숫자가 마지막 줄에 오름차순으로 정렬하세요.

1. **Input/Output 데이터 형태:**
    - **Input**:  숫자가 포함된 텍스트 파일
    - **Output**: 텍스트 파일의 각 줄을 오름차순으로 정렬하여 출력
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `sort`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

sort -t. -n
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
