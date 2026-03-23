---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Sed]]"
  - "[[Linux_Shell_Arrays]]"
  - "[[Linux_Shell_Script]]"
  - "[[Linux_Shell_Arithmetic]]"
source: HackerRank
difficulty:
  - Medium
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 나라 이름이 주어집니다.  각 나라는 **한 줄에 하나씩 입력**됩니다.
> 각 문자열에서 처음 등장하는 대문자 1개를 찾아서 `.`(점)으로 바꾼다
> **모든 나라 이름을 공백(space)으로 구분하여 한 줄로 출력**하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 구성됨
    - **Output**: 각 나라 이름을 변환한 뒤, 공백으로 이어서 한 줄로 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- `readarray` 입력을 배열로 저장
	- `sed` 문자열에서 **첫 번째 대문자만 치환**
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
readarray country
# echo "${country[@]}"

for c in ${country[@]};do
echo "$c"| sed 's/[A-Z]/./'|paste -d '\t' - - -
done
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**
	- `"%s "` = **"문자열 출력 + 뒤에 공백 붙이기"**

---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
readarray country

for c in "${country[@]}"; do
    printf "%s " "$(echo "$c" | sed 's/[A-Z]/./')"
done
```
