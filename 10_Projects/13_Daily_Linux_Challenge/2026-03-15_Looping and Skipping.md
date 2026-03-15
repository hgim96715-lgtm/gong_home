---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Script]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> for loop를 사용하여 **1부터 99까지의 홀수 자연수만 출력**

1. **Input/Output 데이터 형태:**
    - **Input**: 없음 (스크립트 내부에서 숫자 생성)
    - **Output**: `1 ~ 99` 사이의 **홀수 자연수**를 **한 줄씩 출력**
2. **핵심 명령어 & 로직 (Pipeline):**
	- `i<=99` → **99까지 반복**
	- `echo` → 숫자 출력
3. **사용할 주요 옵션 (Flags):**
	- `i+=2` : **2씩 증가 (홀수 생성)**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
for ((i=1; i<=99; i+=2))
do
    echo "$i"
done
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에는 `for i in {1..99}` 로 작성해서 **1~99 전체 숫자가 출력**되었다.
	- 문제는 **홀수만 출력해야 하는데 증가 간격(step)을 고려하지 않은 것**이었다.
-  **새로 알게 된 명령어/옵션:**
	- `start..end..step}` 형태로 사용하며 `..`을 **한 번 더 써서 증가값을 지정**해야 한다.!!!!!! 잊지말자 

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
for i in {1..99..2}; do echo "$i"; done
```
