---
tags:
  - Linux_Test
status: 🟨 힌트참고
related:
  - "[[Stream_Editor]]"
  - "[[Text_Slicing_cut]]"
source: HackerRank
---

# [문제 제목]

## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 텍스트의 각 줄에서 문자를 추출합니다.
> 각 입력 라인에서 **2번째, 7번째 위치의 문자**를 출력합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄의 텍스트 입력
    - **Output**: 각 줄마다 2번째와 7번째 문자를 이어서 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- `substr()` 함수로 지정한 위치의 문자를 추출
3. **사용할 주요 옵션 (Flags):**
	- `awk`

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

awk '{ print substr($0,2,1) substr($0,7,1) }'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- `substr()` 만 쓰면 출력되지 않고 `print` 또는 `printf` 가 필요함
	- `do / done` 은 awk 문법이 아니라는 것을 혼동함

-  **새로 알게 된 명령어/옵션:**
	- 간단한 위치 추출은 **for문 없이도 substr을 바로 이어 붙일 수 있음**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
cut -c2,7
```

-  **`2,7`** (콤마): 2번과 7번 **딱 두 개만** (개별 선택)