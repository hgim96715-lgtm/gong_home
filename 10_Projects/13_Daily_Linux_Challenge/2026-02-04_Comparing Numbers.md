---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[Shell_Scripting_Basics]]"
source:
---
## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 두 정수가 주어졌을 때 X<Y 또는 X > Y 또는 X= Y 확인 

1. **Input/Output 데이터 형태:**
    - **Input**: 각각 하나의 정수를 포함하는 두 줄(그리고각각).
    - **Output**: 다음 조건 중 정확히 하나만 충족해야 합니다.  
		- _X는 Y보다 작다_  
		- _X는 Y보다 크다_  
		- _X는 Y와 같다_
2. **핵심 명령어 & 로직 (Pipeline):**
	- `if / elif / else` 조건문을 사용하여 두 정수를 비교
3. **사용할 주요 옵션 (Flags):**
	- `-eq` :두 정수가 같은지 비교
	- `-ge` : 이상 
	- `-le` : 이하
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

read X
read Y

if [ "$X" -eq "$Y" ]; then
	echo "X is equal to Y"
elif [ "$X" -ge "$Y" ]; then
	echo "X is greater than Y"
elif [ "$X" -le "$Y" ]; then
	echo "X is less than Y"
fi
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에 조건식에서 연산자와 피연산자를 한꺼번에 묶음
	- `"$X -eq $Y"` 전체가 **하나의 문자열**로 인식됨
	- 각 요소는 반드시 공백으로 분리되어야 함
-  **새로 알게 된 명령어/옵션(헷갈렸던 부분 한번 더 정리):**
	- 항상 왼쪽 → 기준(주어), 오른쪽 → 비교 대상, 항상 **왼쪽을 주어**로 읽기
	- `{bash}[ 왼쪽값  연산자  오른쪽값 ]` -> `{bash}[ "$X" -gt "$Y" ]: X가 Y보다 크다 ` 

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
