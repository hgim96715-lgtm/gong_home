---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Sed]]"
source: HackerRank
difficulty:
  - Medium
---
## 문제 분석 & 파이프라인 설계 (Shell logic)

> 각 줄에서 **'thy'라는 단어를 모두 'your'로 바꾸는 작업**을 해야 합니다.
> 대소문자 구분 없이 변환해야 합니다.
> `thy`, `Thy`, `tHy`, `THY` → 전부 `your`로 변경

1. **Input/Output 데이터 형태:**
    - **Input**: 여러 줄의 텍스트
    - **Output**: 각 줄에서 `'thy'`를 `'your'`로 변환한 결과 출력
2. **핵심 명령어 & 로직 (Pipeline):**
	- `sed`를 사용하여 문자열 치환 수행
3. **사용할 주요 옵션 (Flags):**
	- `s` → substitute (치환)
	- `g` → global (한 줄 내 모든 매칭 치환)
	- `i` → ignore case (대소문자 무시)
---
## ✅ 정답 스크립트 (Solution)

```bash
#!/bin/bash
sed 's/thy/your/gi'
```

---
## 오답 노트 & 배운 점 (Retrospective)

- **내가 실수한 부분 (Syntax/Spacing):**

-  **새로 알게 된 명령어/옵션:**


---
## 더 나은 풀이 (One-liner / Efficiency)

```bash
# 숏코딩 / 효율적인 한 줄 명령어
sed 's/\bthy\b/your/gi'
```
