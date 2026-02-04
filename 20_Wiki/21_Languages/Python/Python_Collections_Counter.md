---
aliases:
  - Python Counter
  - 컬렉션
  - 빈도수 세기
  - 데이터 등수
tags:
  - Python
related:
  - "[[Python_Statistics_Module|통계 모듈 (statistics.mode)]]"
  - "[[Python_Builtin_Functions|기본 내장 함수 (sum, len)]]"
  - "[[Python_Dictionaries|딕셔너리 기초 (key-value)]]"
  - "[[00_Python_HomePage]]"
---
##  데이터의 개수를 세는 가장 스마트한 방법 (`Counter`) 

리스트나 문자열에 포함된 요소들의 빈도수를 딕셔너리 형태로 순식간에 계산합니다.

- **문법**: `{python}from collections import Counter`

---
##  주요 활용법

### ① 리스트 요소 빈도 계산

리스트를 `Counter()`에 집어넣기만 하면 "항목: 개수" 형태의 객체가 생성됩니다.

```python
from collections import Counter

data = ['apple', 'banana', 'apple', 'orange', 'banana', 'apple']
counts = Counter(data)

print(counts) 
# 결과: Counter({'apple': 3, 'banana': 2, 'orange': 1})
```

### ② 상위 N개 추출 (`most_common`)  (가장 중요)

가장 많이 등장한 순서대로 데이터를 뽑아줍니다.

- **문법**: `{python}counts.most_common(N)`

```python
# 가장 많이 나온 상위 2개 추출
print(counts.most_common(2))
# 결과: [('apple', 3), ('banana', 2)]
```

### ③ 문자열 분석

문자열을 넣으면 알파벳(또는 단어)이 각각 몇 번 쓰였는지 바로 계산합니다.

```python
text_count = Counter("abracadabra")
print(text_count)
# 결과: Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
```

---
## 요약 및 비교 (Cheat Sheet)

| **기능**     | **사용법**                | **비유**           |
| ---------- | ---------------------- | ---------------- |
| **빈도 측정**  | `Counter(data)`        | **전수 조사 버튼 클릭**  |
| **상위권 추출** | `.most_common(n)`      | **금/은/동 메달 리스트** |
| **요소 확인**  | `.keys()`, `.values()` | **항목과 점수판 분리**   |