---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Arithmetic]]"
source: solvesql
difficulty:
  - Medium
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> `+`, `-`, `*`, `/`, `^` 연산자와 괄호 `()`가 포함된 **수학식 문자열**이 주어집니다.
> 이 수식을 계산한 뒤 **소수점 셋째 자리까지 반올림하여 출력**하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: `+`, `-`, `*`, `/`, `^` 연산자와 **괄호가 포함된 수학식 문자열 1개**
    - **Output**: 수식을 계산한 **결과값을 소수점 셋째 자리까지 반올림하여 출력**
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
read air
res=$(echo "scale=4; $air" |bc)
printf "%.3f\n" $res
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에는 `bc`로 계산하는 것까지만 생각하고 결과를 소수점 셋째 자리까지 반올림하는 방법을 떠올리지 못했다.

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
