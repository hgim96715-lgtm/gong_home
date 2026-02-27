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

> 각 줄에 하나의 숫자가 들어 있는 텍스트 파일이 주어진다.  
> 각 숫자는 정수이거나 소수일 수 있으며, 숫자 외의 다른 문자는 포함되지 않는다.  
> 이때 파일에 포함된 숫자들을 **수치 기준으로 내림차순(Descending Order)** 으로 정렬하여 출력하시오.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 이루어진 텍스트 파일
    - **Output**: 입력으로 주어진 숫자들을 **숫자 크기 기준 내림차순**으로 정렬한 결과
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `sort`
	- `-t'.'` 
	- `-r`
	- `-n`
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

sort -t'.' -rn
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
