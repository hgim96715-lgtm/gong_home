---
tags:
  - Linux_Test
status: 🟧 복습
related:
  - "[[Text_Slicing_cut]]"
  - "[[Stream_Editor]]"
source: HackerRank
---
##  문제 분석 & 파이프라인 설계 (Shell logic)

> 주어진입력된 줄들을 출력합니다
> 각 줄의 문자를 새로운 줄로 출력합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 

```bash
Hello
World
how are you
```

- **Output**

```bash
l
r
w
```

2. **핵심 명령어 & 로직 (Pipeline):**
	- 3번째 문자를 추출하여 출력
3. **사용할 주요 옵션 (Flags):**
	- `awk`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
awk '{print substr($0,3,1)}'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에 `read N` 을 해버려서 `awk` 했을때 `l` 이 안나오는 문제가 생겼다.
	- `read N`은 입력의 첫줄을 미리 읽어버리기때문에 `awk` 는 남아있는 입력만 처리하는 현상이 발생한 것 

-  **새로 알게 된 명령어/옵션:**
	- `read` : 표준 입력에서 **한 줄을 즉시 소비**
	-  `awk` : stdin으로 **남아 있는 라인만 처리**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
cut -c 3
```
