---
tags:
  - Linux_Test
status: 🟧 복습
related:
  - "[[Text_Transformation_tr]]"
  - "[[Shell_Parameter_Expansion]]"
  - "[[Shell_Scripting_Basics]]"
source: HackerRank
---
## 📋 문제 분석 & 파이프라인 설계 (Shell logic)

> 터미널에 입력하기 전, 데이터 흐름(Stream)과 명령어 조합을 먼저 설계합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: STDIN으로 전달되는 **단일 문자 (`Y`, `y`, `N`, `n`)**
    - **Output**: `"YES"` 또는 `"NO"`
2. **핵심 명령어 & 로직 (Pipeline):**
	- read, echo, tr, if/elif
3. **사용할 주요 옵션 (Flags):**
	- `{text}=` : 문자열 비교 연산자 (숫자 비교 아님)
	- `tr 'A-Z' 'a-z'`
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
read STDIN
STDIN=$(echo "$STDIN" | tr 'A-Z' 'a-z')
if [ "$STDIN" = "y" ]; then
	echo "YES"
elif [ "$STDIN" = "n" ]; then
	echo "NO"
fi
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
read c; case${c,,} in y) echo YES ;; n) echo NO ;; esac
```

- 문법 :`{bash}case $변수 in 패턴) 명령어 '' esac`
- `${var,,}` : 모든 문자를 소문자로 
- `${var^^}` : 모든 문자를 대문자로