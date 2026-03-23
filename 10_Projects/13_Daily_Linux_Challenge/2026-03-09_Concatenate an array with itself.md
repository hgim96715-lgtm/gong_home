---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Script]]"
  - "[[Linux_Shell_Arrays]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 이 나라 이름들을 **배열(array)에 저장한 뒤**, 그 배열을 **자기 자신과 두 번 더 이어 붙여** 총 **3번 반복된 배열**을 만듭니다.  
> 그리고 **모든 나라 이름을 공백(space)으로 구분하여 출력**하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: 한 줄에 하나씩 주어지는 **나라 이름들**
    - **Output**: 원래 배열을 3번 이어 붙인 결과를 공백으로 구분하여 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- `readarray`
3. **사용할 주요 옵션 (Flags):**
	- `-t` 줄바꿈 제거 

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

readarray -t countries
echo "${countries[@]}" "${countries[@]}" "${countries[@]}"
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에는 `for`문과 `echo -n`을 사용해 배열을 3번 출력하려고 했다. 하지만 `readarray`로 입력을 받을 때 **각 요소에 줄바꿈(`\n`)이 포함된 상태로 저장**되어 출력 시 나라 이름이 **줄마다 끊어지는 문제**가 발생했다. 또한 `echo -n`을 사용할 경우 **마지막에 불필요한 공백(trailing space)**이 생길 수 있어 채점에서 오답이 될 가능성이 있었다.

-  **새로 알게 된 명령어/옵션:**
	- `readarray -t` : 입력을 배열로 읽을 때 **각 줄의 줄바꿈 문자를 제거**한다.
	- `"${array[@]}"` : Bash에서 **배열 전체 요소를 공백으로 구분해 출력**할 수 있다.


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
