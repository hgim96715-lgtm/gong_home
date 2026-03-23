---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Shell_Arrays]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 줄에 걸쳐 입력되는 국가 이름을 배열에 저장한 뒤, 모든 국가 이름을 **공백으로 구분하여 한 줄로 출력**해야 한다.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 입력되는 **국가 이름 문자열**, 문자 구성: **대소문자 알파벳 + 하이픈(-)**
    - **Output**: 입력받은 국가 이름들을 **공백(space)으로 구분하여 한 줄로 출력**
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
readarray names
echo ${names[@]}
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
names=()

while read name
do
    names+=("$name")
done

echo ${names[@]}
```
