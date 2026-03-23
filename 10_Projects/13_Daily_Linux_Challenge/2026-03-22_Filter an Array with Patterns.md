---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Script]]"
source: HackerRank
difficulty:
  - Medium
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 나라 이름이 한 줄씩 주어집니다.이 나라 이름들을 **배열**에 읽어옵니다.배열에서 **'a' 또는 'A'가 포함된 이름**을 제거합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 각 나라 이름이 한 줄
    - **Output**: **'a' 또는 'A'가 포함된 나라 이름 제거 후** 남은 이름을 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- `{text}=~`  정규식 패턴과 일치하는지 검사
	-  `[[ ]]` 이중 괄호 안에서만 사용 가능
3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash

while read country;do
	if [[ ! "$country" =~ [aA] ]];then
	echo "$country"
	fi
done
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
