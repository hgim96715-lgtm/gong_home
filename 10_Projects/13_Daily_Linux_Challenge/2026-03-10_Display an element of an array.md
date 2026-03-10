---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Arrays]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 나라의 이름이 **각 줄마다 하나씩 주어집니다.**  
> 이 나라 이름들을 **배열(array)에 저장한 뒤**, **지정된 인덱스 위치에 있는 원소를 출력**하세요.  
> 단, **배열의 인덱스는 0부터 시작합니다**

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 입력되는 **국가 이름 목록**
    - **Output**: 입력받은 국가 이름을 **배열에 저장한 뒤, 특정 인덱스 위치(3)에 있는 국가 이름 1개 출력**
2. **핵심 명령어 & 로직 (Pipeline):**
	- `readarray` 명령어를 사용하여 **표준 입력으로 들어오는 여러 줄 데이터를 배열에 저장**

3. **사용할 주요 옵션 (Flags):**
	- `readarray`표준 입력으로 들어온 **여러 줄의 데이터를 배열에 저장**하는 Bash 명령어
	- `"${countries[3]}"` 배열 `countries`의 **3번 인덱스(네 번째 값)** 를 출력
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

readarray countries
echo "${countries[3]}"
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
