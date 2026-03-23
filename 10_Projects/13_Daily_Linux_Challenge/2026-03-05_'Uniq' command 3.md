---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Awk]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 텍스트 파일이 주어질 때, **연속해서 반복되는 각 줄의 반복 횟수**를 세어 출력하시오.
> 출력은 각 줄에 대해 '반복횟수 줄내용'형식으로 하며, 반복 횟수와 줄 내용 사이는 **공백 하나**로 구분한다.  
> 출력 결과에는 **앞뒤 공백이 없어야 한다.**

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 이루어진 **텍스트 파일**
    - **Output**: 각 연속된 줄 묶음에 대해
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
awk 'BEGIN{}
{
    k = tolower($0)
    if (NR == 1) {
        pk = k; pl = $0; c = 1
    } else if (k == pk) {
        c++
    } else {
        print c, pl
        pk = k; pl = $0; c = 1
    }
}
END {
    print c, pl
}'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- `tr '[A-Z]' '[a-z]' | uniq -c | awk '{$1=$1; print}'`

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
```
