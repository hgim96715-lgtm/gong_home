---
aliases:
  - Cosine
  - TF
  - IDF
tags:
  - Sklearn
related:
  - "[[00_Skelen_HomePage]]"
  - "[[Sklearn_TF_IDF]]"
  - "[[CS_Vector_Matrix]]"
  - "[[Numpy_Sparse_Matrix]]"
---
# Cosine Similarity (코사인 유사도)


---

## 1. 핵심 아이디어 한 줄

**"같은 단어를 많이 쓸수록 같은 방향을 가리킨다 → 유사도 높다"**

---

## 2. 단계별로 이해하기

### STEP 1. 문서를 벡터로 변환 (TF-IDF)

```
단어 목록:    [project, budget, cooking, management, dance]
NCS 문서:    [ 0.5,    0.5,    0,        0.7,        0   ]
O*NET 관리직: [ 0.4,    0.3,    0,        0.6,        0   ]
요리책:       [ 0,      0,      0.9,      0,          0.8 ]
```

### STEP 2. 벡터를 화살표로 시각화

```
project축 ↑
          │   /← NCS [0.5, 0.7]
          │  /
          │ /← O*NET [0.4, 0.6]
          │/
          └──────────────→ management축

NCS와 O*NET 관리직: 거의 같은 방향 → 각도 작음 → 유사도 높음


project축 ↑
          │← NCS [0.5, 0]
          │
          │
          └──────────────→ cooking축
                        ↗ 요리책 [0, 0.9]

NCS와 요리책: 직각(90°) → 유사도 0
```

### STEP 3. 각도 → cos 값 → 유사도

```
θ (각도)   cos(θ)   의미
─────────────────────────────────────
0°         1.0      완전히 같은 방향 = 유사도 최고
45°        0.7      비슷한 방향
90°        0.0      직각 = 공통 단어 없음
180°      -1.0      반대 방향 (TF-IDF에서는 안 나옴)
```

---

## 3. 코드 완전 분해

```python
from sklearn.metrics.pairwise import cosine_similarity

scores = cosine_similarity(matrix[0:1], matrix[1:]).flatten()
```

### matrix[0:1] vs matrix[1:]

```python
# matrix shape: (1017, 4867)
#                ↑문서수  ↑단어수

matrix[0]    # shape: (4867,)       ← 1차원 벡터 하나
matrix[0:1]  # shape: (1, 4867)     ← 2차원 행렬 (1행짜리)
matrix[1:]   # shape: (1016, 4867)  ← 2차원 행렬 (O*NET 1016개)

# cosine_similarity는 2차원만 받음 → 반드시 matrix[0:1] 사용
```

### cosine_similarity 결과

```python
result = cosine_similarity(matrix[0:1], matrix[1:])
# result shape: (1, 1016)
# NCS 1개 vs O*NET 1016개 → 유사도 1016개

result.flatten()
# shape: (1016,) ← 1차원으로 펼치기
# [0.31, 0.08, 0.00, 0.29, ...]
#   ↑1번   ↑2번  ↑3번
```

---

## 4. 직접 손계산 (단어 3개짜리)

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# 단어: [project, budget, cooking]
ncs    = [[0.5, 0.5, 0.0]]   # project, budget 씀
onet_a = [[0.7, 0.7, 0.0]]   # project, budget 씀 (비슷)
onet_b = [[0.0, 0.0, 1.0]]   # cooking만 씀 (완전 다름)

print(cosine_similarity(ncs, onet_a))  # [[1.0]] ← 같은 방향
print(cosine_similarity(ncs, onet_b))  # [[0.0]] ← 직각

# 한번에 계산
all_onet = [[0.7, 0.7, 0.0],
            [0.0, 0.0, 1.0]]

scores = cosine_similarity(ncs, all_onet).flatten()
print(scores)  # [1.0, 0.0]
```

---

## 5. 실제 과제 전체 코드

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

corpus = [ncs_english] + onet["Description"].fillna("").tolist()
# corpus[0]  = NCS 번역 텍스트
# corpus[1~] = O*NET 1016개 직업 설명

vectorizer = TfidfVectorizer()
matrix = vectorizer.fit_transform(corpus)
# shape: (1017, 4867)

scores = cosine_similarity(matrix[0:1], matrix[1:]).flatten()
# shape: (1016,)

onet["유사도"] = scores
top10 = onet.sort_values("유사도", ascending=False).head(10)
print(top10[["Title", "유사도"]])
```

---

## 6. 유사도 점수 해석

```
0.4 이상  → 매우 유사  → 문항 1 후보 1순위
0.2~0.4   → 어느 정도  → 문항 1 후보 2~3순위
0.2 미만  → 관련 낮음  → 제외

⚠️ 절대적 기준 아님. 상위 N개 추출 방식이 더 실용적
```

---

## 7. 자주 하는 실수

```python
# ❌ 잘못된 코드
cosine_similarity(matrix[0], matrix[1:])
# → ValueError: 1차원 배열은 못 받음

# ✅ 올바른 코드
cosine_similarity(matrix[0:1], matrix[1:])

# ❌ 잘못된 이해: "유사도 0 = 반대 의미"
# ✅ 올바른 이해: "유사도 0 = 공통 단어 없음 (직각)"
```

---
