---
tags:
  - Linux_Test
status: 🟩 해결
related:
  - "[[00_Linux_Challenge_DashBoard]]"
source: HackerRank
difficulty:
  - Easy
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 여러 개의 **국가 이름이 한 줄씩 입력**됩니다.  
> 이 국가 이름들을 **배열(array)에 저장한 뒤**, 배열의 **특정 위치 구간을 슬라이싱**하여 출력해야 합니다.
> 배열에서 **인덱스 3부터 인덱스 7까지(둘 다 포함)** 의 요소만 출력하세요.  
> 배열의 **인덱스는 0부터 시작**합니다.
> 출력할 때는 **각 국가 이름을 공백(space)으로 구분하여 한 줄로 출력**합니다.

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄로 입력되는 **국가 이름 리스트**
    - **Output**: 배열에서 **인덱스 3부터 7까지의 요소**만 출력
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
readarray names
echo ${names[@]:3:5}
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
