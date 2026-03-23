---
tags:
  - Linux_Test
related:
  - "[[Linux_Awk]]"
  - "[[00_Linux_Challenge_DashBoard]]"
source: HackerRank
---
##  문제 분석 & 파이프라인 설계 (Shell logic)

>탭으로 구분된 여러 열로 구성된 파일(tsv 형식)이 주어졌을 때, 처음 세 필드를 출력하세요.

1. **Input/Output 데이터 형태:**
    - **Input**: 텍스트로만 구성된 탭으로 구분된 파일
    - **Output**: 출력은 N줄로 구성되어야 합니다. 입력의 각 줄에 대해 처음 세 필드를 출력
2. **핵심 명령어 & 로직 (Pipeline):**

3. **사용할 주요 옵션 (Flags):**

---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
awk -F'\t' '{print $1,$2,$3}'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**
	- `$1\t$2\t$3` 처럼 **`\t`를 필드 사이에 직접 쓰려고 했다**
	- awk는 **각 줄을 자동으로 순회**하므로 `for` 문이 필요 없다
-  **새로 알게 된 명령어/옵션:**
	- `-F'\t'` : 입력 필드 구분자를 **탭(tab)** 으로 명시
	- `print` 는 필드 사이에 **OFS(Output Field Separator)** 를 자동으로 삽입
		- 필요하면 `BEGIN { OFS="\t" }` 로 변경 가능

---
## 더 나은 풀이 (One-liner / Efficiency) + 동일동작 다른예 

```bash
#!/bin/bash
awk 'BEGIN { FS=OFS="\t" } { print $1, $2, $3 }'
```
