---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Data_Statistics]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 파이프(`|`)로 구분된(weather data) 텍스트 파일이 주어진다. 이 파일에는 **헤더(header)가 없다**.
> **2번째 컬럼(1월 평균 기온)** 을 기준으로 **내림차순(descending order)** 으로 파일 전체를 정렬하라

1. **Input/Output 데이터 형태:**
    - **Input**:  여러 줄로 구성된 텍스트 파일
    - **Output**: **1월 평균 기온(2번째 컬럼)** 기준으로,  **높은 값 → 낮은 값 순서**로 정렬된 결과 출력
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `sort`
	- `-t'|'`
	- `-rn`
	- `-k2`
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

sort -t'|' -k2 -rn
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
