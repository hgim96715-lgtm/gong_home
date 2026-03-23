---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Awk]]"
source: HackerRank
difficulty:
  - Medium
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 개의 **정수**가 주어질 때,  이 정수들의 **평균(average)** 을 계산하고 **소수점 셋째 자리까지 반올림하여 출력**하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: 첫 번째 줄에 **정수 N**이 주어집니다.다음 **N개의 줄**에는 각각 **정수 하나씩** 주어집니다.
    - **Output**: 입력된 **N개의 정수 평균값**
2. **핵심 명령어 & 로직 (Pipeline):**
	- `printf "%.3f"`로 **소수점 셋째 자리까지 출력**
	- `awk`를 사용하여 **각 줄의 숫자를 순회**
3. **사용할 주요 옵션 (Flags):**
	- `END {}` : 모든 입력 처리가 끝난 후 실행되는 블록
	- `"%.3f"` : **소수점 셋째 자리까지 출력**
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
read num

awk '
{
    total += $1
    count++
}
END {
    printf "%.3f", total/count
}
'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- `awk`에서 `$n`은 **n번째 컬럼(field)** 을 의미하며 변수 `num`을 의미하는 것이 아니라는 점을 헷갈렸다.
	- 내가 awk를 잘 이해 하지 못했던게 이 헷갈림 때문이었음을 알게되었다.
-  **새로 알게 된 명령어/옵션:**

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
