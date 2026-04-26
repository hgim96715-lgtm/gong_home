---
aliases:
  - CountVectorizer
  - 단어 빈도 행렬
  - BOW
  - Bag OF Words
tags:
  - Sklearn
related:
  - "[[00_Skelen_HomePage]]"
  - "[[Sklearn_TF_IDF]]"
  - "[[Sklearn_Cosine_Similarity]]"
---

# Sklearn_CountVectorizer — 단어 빈도 행렬

## 한 줄 요약

```
텍스트를 단어 등장 횟수 행렬로 변환
TF-IDF 의 기초 / 단순하지만 빠름

CountVectorizer → 등장 횟수 (정수)
TfidfVectorizer → 중요도 점수 (0~1 실수)
```

---

---

# ① CountVectorizer 개념

```
Bag of Words (BOW) 모델
"단어 순서 무시, 등장 횟수만 셈"

"I love Python and Python loves me"
→ {"I":1, "love":1, "Python":2, "and":1, "loves":1, "me":1}

장점:
  빠름 / 단순 / 이해하기 쉬움
단점:
  단어 순서 무시
  "not good" vs "good" 구분 못함
  흔한 단어(the, is)에 높은 값 → TF-IDF 로 보완 필요
```

---

---

# ② 기본 사용법 ⭐️

```python
from sklearn.feature_extraction.text import CountVectorizer

corpus = [
    "project planning and management",
    "budget control and risk management",
    "overseas business coordination"
]

vectorizer = CountVectorizer()
matrix = vectorizer.fit_transform(corpus)

print(matrix.shape)    # (3, 단어수)
print(matrix.toarray())
# [[0, 0, 0, 0, 0, 1, 0, 1, 1, 0]   ← 문서1: 각 단어 등장 횟수
#  [1, 1, 0, 1, 0, 1, 0, 0, 0, 1]   ← 문서2
#  [0, 0, 1, 0, 1, 0, 1, 0, 0, 0]]  ← 문서3
```

---

---

# ③ vocabulary_ — 어휘 사전 ⭐️

```python
vectorizer.fit(corpus)

# 단어 → 인덱스 딕셔너리
print(vectorizer.vocabulary_)
# {'and': 0, 'budget': 1, 'business': 2, 'control': 3,
#  'coordination': 4, 'management': 5, 'overseas': 6,
#  'planning': 7, 'project': 8, 'risk': 9}

# 인덱스 → 단어 (알파벳 정렬)
print(vectorizer.get_feature_names_out())
# ['and' 'budget' 'business' 'control' 'coordination'
#  'management' 'overseas' 'planning' 'project' 'risk']

# 사전 크기
print(len(vectorizer.vocabulary_))  # 10
```

```
vocabulary_ 특징:
  알파벳 오름차순 정렬
  인덱스 = 행렬의 열 번호
  대소문자 기본적으로 lowercase 처리됨
```

---

---

# ④ get_feature_names_out() — 단어 목록

```python
# 단어 목록 = 행렬의 열 이름
features = vectorizer.get_feature_names_out()

# DataFrame 으로 보기 좋게
import pandas as pd
df = pd.DataFrame(
    matrix.toarray(),
    columns=features,
    index=["문서1", "문서2", "문서3"]
)
print(df)

#       and  budget  business  control  coordination  management  overseas  planning  project  risk
# 문서1   1       0         0        0             0           1         0         1        1     0
# 문서2   1       1         0        1             0           1         0         0        0     1
# 문서3   0       0         1        0             1           0         1         0        0     0
```

---

---

# ⑤ 주요 파라미터

```python
CountVectorizer(
    max_features=1000,     # 상위 N개 단어만
    stop_words='english',  # 불용어 제거
    ngram_range=(1, 2),    # unigram + bigram
    min_df=2,              # 최소 등장 문서 수
    max_df=0.9,            # 최대 등장 비율
    binary=False,          # True 이면 등장 여부만 (0/1)
    lowercase=True,        # 소문자 변환 (기본값)
)
```

## binary 모드

```python
# 등장 횟수 대신 등장 여부만 (0 또는 1)
vectorizer = CountVectorizer(binary=True)
matrix = vectorizer.fit_transform(["Python Python is great"])
print(matrix.toarray())
# [[1, 1, 1]]  ← Python 이 2번이어도 1로 표시
```

## ngram_range (1,2) 예시

```python
vectorizer = CountVectorizer(ngram_range=(1, 2))
matrix = vectorizer.fit_transform(["project management is great"])

print(vectorizer.get_feature_names_out())
# ['great', 'is', 'is great', 'management', 'management is',
#  'project', 'project management']
# ↑ 1개짜리 + 2개짜리 조합 전부
```

---

---

# ⑥ CountVectorizer vs TfidfVectorizer 비교 ⭐️

|구분|CountVectorizer|TfidfVectorizer|
|---|---|---|
|값|정수 (등장 횟수)|실수 (0~1 점수)|
|흔한 단어 처리|높게 평가 (문제)|IDF 로 낮춤|
|속도|더 빠름|약간 느림|
|용도|빠른 탐색, 기초 분석|유사도 계산, NLP|
|메모리|희소 행렬|희소 행렬|

```python
# 같은 텍스트로 비교
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer

corpus = ["good good good", "bad bad", "good bad"]

cv = CountVectorizer()
tv = TfidfVectorizer()

print("Count:", cv.fit_transform(corpus).toarray())
# [[3, 0], [0, 2], [1, 1]]  ← "good" 이 3번 → 높은 값
# bad  good

print("TF-IDF:", tv.fit_transform(corpus).toarray().round(2))
# [[0.  1. ], [1.  0. ], [0.7 0.7]]  ← 균형잡힘
```

---

---

# ⑦ 실전 패턴

## 카운트 행렬 → 유사도

```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.metrics.pairwise import cosine_similarity

corpus = [
    "data engineering pipeline",
    "data pipeline processing",
    "machine learning model"
]

vectorizer = CountVectorizer()
matrix = vectorizer.fit_transform(corpus)

sim = cosine_similarity(matrix)
print(sim.round(2))
# [[1.   0.63 0.  ]
#  [0.63 1.   0.  ]
#  [0.   0.   1.  ]]
# ← 문서1과 문서2가 0.63으로 유사 (data, pipeline 공유)
```

## 새 문서 비교

```python
train = ["project management", "risk control", "budget planning"]
new_doc = ["project budget control"]

vectorizer = CountVectorizer()
train_matrix = vectorizer.fit_transform(train)

# 새 문서는 transform 만 (fit 하면 안 됨)
new_matrix = vectorizer.transform(new_doc)

sim = cosine_similarity(new_matrix, train_matrix)
print(sim)
# [[0.  0.5 0.5]]
# ← 새 문서가 risk control, budget planning 과 유사
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|흔한 단어가 너무 높은 값|등장 횟수만 반영|TfidfVectorizer 사용|
|새 데이터에 fit_transform|vocabulary 달라짐|`transform` 만 사용|
|한글 단어가 분리 안 됨|기본 토크나이저 영어 기준|`analyzer='char'` 또는 KoNLPy|
|단어 순서 무시됨|BOW 모델 한계|bigram `ngram_range=(1,2)` 사용|