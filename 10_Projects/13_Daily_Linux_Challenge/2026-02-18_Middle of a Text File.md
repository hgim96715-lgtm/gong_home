---
tags:
  - Linux_Test
status: 🟧 복습
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[File_Content_Viewing]]"
  - "[[Linux_Stream_Editor]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 주어진 텍스트 파일의 12번째 줄부터 22번째 줄까지(양쪽 줄 포함)를 표시합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트 파일
    - **Output**:  12번째 줄부터 22번째 줄까지(양쪽 줄 포함)
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
tail -n +12 | head -n 11
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- `head`라는 이름 때문에 항상 명령어의 시작이나 파이프의 앞부분에 위치해야 한다고 생각했다.
	- **12번째 줄부터** 보고 싶다면, 먼저 `tail`을 써서 **앞의 1~11번째 줄을 잘라내고(버리고)** 남은 데이터를 `head`에게 넘겨줘야 한다는 **순서의 로직**을 놓쳤다.

-  **새로 알게 된 명령어/옵션:**
	- `tail -n +숫자`: 뒤에서부터가 아니라, **앞에서부터 해당 숫자만큼 건너뛰고 시작**하는 기능.

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
sed -n 12,22p
```
