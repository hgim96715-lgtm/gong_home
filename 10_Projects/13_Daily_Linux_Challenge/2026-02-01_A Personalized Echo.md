---
tags:
  - Linux_Test
status: 🟨 힌트참고
related:
  - "[[Shell_Input_Output]]"
source: HackerRank
---

## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 터미널에 입력하기 전, 데이터 흐름(Stream)과 명령어 조합을 먼저 설계합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 표준 입력(stdin)으로 들어오는 **한 줄의 문자열**
    - **Output**: 표준 출력(stdout)으로 출력되는 **한 줄의 문자열**
2. **핵심 명령어 & 로직 (Pipeline):**
	- `echo` 명령어로 문자열과 변수를 결합하여 출력
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
read name
echo "Welcome $name"
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- Bash에서는 **표준 입력을 명시적으로 받아야 한다**는 점을 놓침
-  **새로 알게 된 명령어/옵션:**
	- `read`-> 표준 입력(stdin)으로부터 한 줄을 읽어 변수에 저장하는 Bash 내장 명령어

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
