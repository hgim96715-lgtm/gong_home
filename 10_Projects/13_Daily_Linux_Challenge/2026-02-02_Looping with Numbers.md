---
tags:
  - Linux_Test
status: 🟨 힌트참고
related:
  - "[[Shell_Scripting_Basics]]"
source: HackerRank
---
## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> _for_ 루프를 사용하여 1에서 50까지의 자연수를 출력하세요

1. **Input/Output 데이터 형태:**
    - **Input**: for루프
    - **Output**: 1부터 50까지의 **자연수를 한 줄에 하나씩 출력**
2. **핵심 명령어 & 로직 (Pipeline):**
	- for 루프 
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
for ((i=1; i<=50; i++)); do
	echo "$i"
done
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에 **Python의 `range(51)` 문법을 쉘 스크립트에서도 사용할 수 있다고 착각함**
	- 쉘에서는 숫자 범위를 표현할 때 `{1..50}` 또는 `for ((i=1; i<=50; i++))` 형태를 사용해야 함

-  **새로 알게 된 명령어/옵션:**
	- `for ((i=1; i<=50; i++))` : C 스타일 for 루프 문법

>Python의 `range()`는 쉘에 없다 — 쉘은 `{}` 또는 `(( ))`로 범위를 만든다.

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
count=1
while [ "$count" -le 50 ]; do
	echo "$count"
	count=$((count + 1))
done
```

- while문 
