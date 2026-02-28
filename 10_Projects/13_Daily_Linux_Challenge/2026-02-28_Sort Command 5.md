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

> 미국 도시의 기온 정보를 담은 **TSV(Tab-Separated Values, 탭 구분) 형식 파일**이 주어집니다.
> 첫 번째 컬럼: 도시 이름, 두 번째 ~ 다섯 번째 컬럼: 1월(Jan), 2월(Feb), 3월(Mar), 4월(Apr) 평균 기온
> **1월 평균 기온(January, 두 번째 컬럼)** 을 기준으로 **내림차순(높은 순)** 으로 테이블의 행을 정렬하세요.
>  정렬 후 행 전체를 그대로 출력하면 됩니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트 파일의 각 행에 **도시 이름 + 1~4월 평균 기온**이 탭(`\t`)으로 구분되어 있음
    - **Output**: 1월 평균 기온을 기준으로 내림차순 정렬된 **도시와 월별 기온 행 전체**
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `sort`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
sort -t$'\t' -k2 -rn
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
