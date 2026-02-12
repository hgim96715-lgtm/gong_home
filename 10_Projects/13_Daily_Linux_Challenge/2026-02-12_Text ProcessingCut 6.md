---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[Text_Slicing_cut]]"
  - "[[Stream_Editor]]"
source: HackerRank
---

# [문제 제목]

## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 열세 번째 위치부터 끝까지 문자를 출력하세요.

1. **Input/Output 데이터 형태:**
    - **Input**:  텍스트 파일
    - **Output**: 13번째 위치부터 마지막 ​​줄까지의 문자를 출력
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**
	- `cut`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
cut -c 13-
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 다른 방법 
awk '{print substr($0,13)}'
```
