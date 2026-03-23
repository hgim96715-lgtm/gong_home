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

> 탭(`\t`)으로 구분된 날씨 데이터 파일이 주어진다.이 파일에는 헤더가 없으며, 각 행은 하나의 도시와 해당 도시의 월별 평균 기온 정보를 담고 있다.
> **두 번째 컬럼(1월 평균 기온)** 을 기준으로  전체 데이터를 **오름차순(ascending order)** 으로 정렬해야 한다.

1. **Input/Output 데이터 형태:**
    - **Input**: 헤더가 없는 **TSV(Tab Separated Values)** 파일
    - **Output**: 단, **2번째 컬럼(1월 평균 기온)을 기준으로 오름차순 정렬**
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `sort`
	- `-t $'\t'` 필드 구분자를 **탭(tab)** 으로 지정
	- `-k2` **두 번째 컬럼**을 정렬 기준
	- `-n` 문자열이 아닌 **숫자 기준 정렬**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

sort -t$'\t' -k2 -n
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
