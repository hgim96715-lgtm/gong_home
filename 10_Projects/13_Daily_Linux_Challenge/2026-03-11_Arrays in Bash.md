---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Arrays]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 나라 이름이 **한 줄에 하나씩 주어집니다.** 입력된 나라 이름들을 **배열(array)에 저장한 뒤**
>  그 **배열에 들어있는 요소 개수**를 출력하는 것

1. **Input/Output 데이터 형태:**
    - **Input**: 나라 이름들이 **여러 줄로 입력됩니다.**
    - **Output**: 배열에 들어있는 **요소의 개수(나라 개수)** 를 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- `readarray`를 사용하여 **표준 입력으로 들어오는 여러 줄 데이터를 배열에 저장**
3. **사용할 주요 옵션 (Flags):**
	- `readarray` : 표준 입력을 **한 줄씩 읽어 배열에 저장**
	- `${#countries[@]}` : **배열 요소 개수 반환**
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
readarray countries

echo "${#countries[@]}"
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
