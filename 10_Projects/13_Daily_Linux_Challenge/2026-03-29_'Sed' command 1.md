---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Sed]]"
source: programmers
difficulty:
  - Medium
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

>  `"the"`라는 단어의 **첫 번째 등장만**  `"this"`로 바꾸기
>  **대소문자를 구분함** (case-sensitive)
>   `the`만 바꾸고 `The`, `THE`는 바꾸지 않음

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트 파일
    - **Output**: 각 줄에서 `"the"`의 **첫 번째 등장만 `"this"`로 변환한 결과 출력**
2. **핵심 명령어 & 로직 (Pipeline):**
	- **첫 번째만 치환** (`g` 사용 X)
3. **사용할 주요 옵션 (Flags):**
	- `\b` 단어 경계 (word boundary)
	- `g 없음` → 첫 번째만 치환
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

sed 's/\bthe\b/this/'
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
