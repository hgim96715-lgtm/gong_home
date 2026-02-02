---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[Shell_Arithmetic]]"
  - "[[Shell_Input_Output]]"
source: HackerRank
---

# [문제 제목]

## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 두 정수가 주어졌을 때,그리고두 수의 합, 차, 곱, 몫을 구하세요

1. **Input/Output 데이터 형태:**
    - **Input**: 두 정수 (한 줄 또는 두 줄)
    - **Output**: 덧셈, 뺄셈, 곱셈, 나눗셈 결과 (정수)
2. **핵심 명령어 & 로직 (Pipeline):**
	- `$(())` : 산술계산
	- `read` : 사용자 입력
	- `echo` : 계산 결과 출력 
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

read x
read y

sum=$(($x+$y))
dif=$(($x-$y))
pro=$(($x*$y))
quo=$(($x/$y))

echo $sum
echo $dif
echo $pro
echo $quo
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
#!/bin/bash
read x y

echo "합: $(($x + $y))"
echo "차: $(($x - $y))"
echo "곱: $(($x * $y))"
echo "몫: $(($x / $y))"

```

- 한줄에 두 수를 입력받을수도 있다
