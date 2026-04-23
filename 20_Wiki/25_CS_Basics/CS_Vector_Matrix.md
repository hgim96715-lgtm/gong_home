---
aliases:
  - 벡터
  - 행렬
tags:
  - CS
related:
  - "[[00_CS_HomePage]]"
  - "[[Numpy_Array_Basics]]"
  - "[[Numpy_Sparse_Matrix]]"
  - "[[Sklearn_TF_IDF]]"
  - "[[Sklearn_Cosine_Similarity]]"
---

# 벡터와 행렬 (Vector & Matrix)

> 이 개념을 모르면 TF-IDF, cosine_similarity, numpy가 전부 블랙박스가 됨 sklearn 이해의 핵심 전제지식

---

## 1. 벡터 (Vector)

### 벡터란?

**숫자를 순서대로 나열한 것**

```
[3, 1, 4]         ← 3개짜리 벡터 (3차원)
[0.29, 0.64, 0]   ← 3개짜리 벡터
[1, 0, 0, 0, 1]   ← 5개짜리 벡터 (5차원)
```

### 왜 쓰나?

**"어떤 것"을 숫자로 표현하기 위해서**

```
예시: 사람을 벡터로 표현
[키, 몸무게, 나이]
[175, 70, 25]  ← 이 사람은 키175, 몸무게70, 나이25

예시: 문서를 벡터로 표현 (TF-IDF)
단어 목록: [project, budget, cooking, management, dance]
NCS 문서:  [0.29,   0.36,   0,       0.64,       0    ]
요리 문서:  [0,      0,      0.91,    0,          0    ]
           ↑ 겹치는 단어가 없음 → 유사도 0
```

### 벡터의 방향과 크기

```
2차원으로 그리면:

      ↑ y
   2  │    *(2,2)
      │   /
   1  │  /
      │ /
      └──────→ x
         1  2

벡터 [2, 2]는 오른쪽 위 대각선 방향을 가리킴
벡터 [1, 0]은 오른쪽 방향을 가리킴
```

---

## 2. 행렬 (Matrix)

### 행렬이란?

**벡터를 여러 개 쌓은 것 = 2차원 표**

```
벡터 3개:
[1, 2, 3]
[4, 5, 6]
[7, 8, 9]

이걸 쌓으면 행렬:
┌         ┐
│ 1  2  3 │  ← 1번 행 (row)
│ 4  5  6 │  ← 2번 행
│ 7  8  9 │  ← 3번 행
└         ┘
  ↑  ↑  ↑
  1  2  3번 열(column)

shape = (3, 3)  ← (행 수, 열 수)
```

### TF-IDF 행렬이 뭔지 이제 이해

```
문서가 3개, 단어가 5개면:

          project  budget  cooking  management  dance
NCS문서  [ 0.29,   0.36,   0,       0.64,       0   ]  ← 1번 행
O*NET1  [ 0.14,   0.20,   0,       0.20,       0   ]  ← 2번 행
요리책   [ 0,      0,      0.91,    0,          0.8 ]  ← 3번 행

shape = (3, 5)  ← (문서 수, 단어 수)

실제 과제에서:
shape = (1017, 4867)  ← (문서 1017개, 단어 4867개)
                            NCS 1개 + O*NET 1016개
```

---

## 3. 1차원 vs 2차원 - matrix[0] vs matrix[0:1]

```python
import numpy as np
arr = np.array([
    [1, 2, 3],   # 0번 행
    [4, 5, 6],   # 1번 행
    [7, 8, 9],   # 2번 행
])

print(arr[0])    # [1, 2, 3]       shape: (3,)    ← 1차원
print(arr[0:1])  # [[1, 2, 3]]     shape: (1, 3)  ← 2차원
```

```
arr[0]   →  [1, 2, 3]      ← 그냥 숫자 나열 (1차원)
arr[0:1] → [[1, 2, 3]]     ← 행렬 안에 있는 한 행 (2차원)

차이: 대괄호 겹침 여부
[1, 2, 3]    = 1차원 벡터
[[1, 2, 3]]  = 1행짜리 2차원 행렬
```

**cosine_similarity가 왜 2차원을 요구하냐:**

```python
# cosine_similarity는 "행렬 vs 행렬" 비교
# 행렬은 2차원이어야 함

cosine_similarity(matrix[0:1], matrix[1:])
#                 (1, 4867)   (1016, 4867)
#                 ↑ 2차원      ↑ 2차원
# → 결과: (1, 1016) → .flatten() → (1016,)
#   NCS와 O*NET 1016개 각각의 유사도 점수
```

---

## 4. 코사인 유사도 - 각도로 이해

### 핵심 아이디어

**두 벡터가 같은 방향을 가리키면 유사하다**

```
2차원으로 예시:

      ↑
      │   /← NCS벡터 [2, 3]
      │  /
      │ /← O*NET벡터 [1, 2]
      │/________→

두 벡터가 거의 같은 방향 → 각도 작음 → 유사도 높음
```

```
      ↑
      │← NCS벡터 [0, 3]
      │
      │
      └──────────→ O*NET벡터 [3, 0]

두 벡터가 직각(90°) → 공통 단어 없음 → 유사도 0
```

### 단어로 이해 (더 직관적)

```
NCS 벡터:    [project=0.29, management=0.64, cooking=0,   dance=0  ]
O*NET A 벡터: [project=0.14, management=0.20, cooking=0,   dance=0  ]
O*NET B 벡터: [project=0,    management=0,    cooking=0.91, dance=0.8]

NCS vs O*NET A: project, management 단어가 겹침 → 같은 방향 → 유사도 높음
NCS vs O*NET B: 겹치는 단어 없음 → 완전히 다른 방향 → 유사도 0
```

### 공식 (참고만)

```
cos(θ) = (A · B) / (|A| × |B|)

A · B = 각 위치 숫자끼리 곱해서 더하기 (내적)
|A|   = 벡터의 크기 (길이)

결과: -1 ~ 1 사이
TF-IDF는 값이 항상 0 이상이라 → 0 ~ 1 사이만 나옴
```

---

## 5. 실제 손계산 예시 (단어 3개짜리)

```python
# 단어 3개: [project, budget, cooking]
# NCS:      [0.5,     0.5,    0      ]
# O*NET A:  [0.7,     0.7,    0      ]  ← 비슷할 것
# O*NET B:  [0,       0,      1.0    ]  ← 달라야 함

import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

ncs   = [[0.5, 0.5, 0  ]]
onetA = [[0.7, 0.7, 0  ]]
onetB = [[0,   0,   1.0]]

print(cosine_similarity(ncs, onetA))  # [[1.0]]  ← 완전 유사
print(cosine_similarity(ncs, onetB))  # [[0.0]]  ← 전혀 무관
```

---

## 6. 정리

```
벡터  = 숫자를 순서대로 나열한 것 (문서를 숫자로 표현)
행렬  = 벡터를 여러 개 쌓은 것 (모든 문서를 한번에 표현)

TF-IDF 행렬 = 각 문서를 벡터로 변환해서 쌓은 행렬
              shape: (문서 수, 단어 수)

cosine_similarity = 두 벡터의 방향이 얼마나 같은지
                    같은 단어를 많이 쓸수록 → 방향 비슷 → 점수 높음
```

---

## 관련 노트

- [[Numpy_Array_Basics]] ← 벡터/행렬을 numpy로 다루기
- [[Numpy_Sparse_Matrix]] ← 0이 많은 행렬 효율적으로 저장하기
- [[Sklearn_TF_IDF]] ← 문서를 벡터로 변환
- [[Sklearn_Cosine_Similarity]] ← 벡터 유사도 계산