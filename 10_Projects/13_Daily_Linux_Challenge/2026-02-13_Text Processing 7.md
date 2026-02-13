---
tags:
  - Linux_Test
status: 🟧 복습
related:
  - "[[Text_Slicing_cut]]"
source: HackerRank
---
## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 주어진 문장의 네 번째 단어를 찾아 표시하세요. 
> 단어 사이에는 공백(' ')만 있다고 가정합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트 파일
    - **Output**: 각 입력 문장의 네 번째 단어를 찾아 출력
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
cut -d ' ' -f4
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- 처음에 `awk`로 접근
	- 코드의 문제는 **4번째 단어가 없을 때 줄 전체를 출력**하도록 짰는데, 실제 기대 출력과 달랐습니다. `how are you`(3단어)에 대해 `how are you`를 출력했지만, 정답은 **빈 줄**이어야 했다

-  **새로 알게 된 명령어/옵션:**
	- `cut -d ' ' -f4`
	- `-d ' '`구분자(delimiter)를 공백으로 지정
	- `-f4`4번째 필드(단어)를 추출


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
