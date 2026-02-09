---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[Stream_Editor]]"
source: HackerRank
---

# [문제 제목]

## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

>문자열에서 2번째 위치부터 7번째 위치까지(양 끝 포함) 문자를 출력 하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: 표준 입력(stdin)으로 들어오는 문자열
    - **Output**: 입력 문자열의 **2번째 문자부터 7번째 문자까지 출력**
2. **핵심 명령어 & 로직 (Pipeline):**
	- `awk`
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
awk '{print substr($0,2,6)}'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
cut -c 2-7
```
