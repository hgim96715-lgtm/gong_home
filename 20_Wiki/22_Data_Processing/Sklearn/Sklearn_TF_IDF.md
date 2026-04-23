---
aliases:
  - TF
  - IDF
  - TF-IDF
  - DF
tags:
  - Sklearn
related:
  - "[[00_Skelen_HomePage]]"
---
# TF-IDF

## 개념

|용어|풀이|의미|
|---|---|---|
|TF|Term Frequency|하나의 문서에서 특정 단어가 등장하는 횟수|
|DF|Document Frequency|특정 단어가 등장하는 문서의 수|
|IDF|Inverse DF|DF의 역수|
|TF-IDF|TF × IDF|최종 중요도 점수 (0 ~ 1)|

---

## IDF가 왜 필요한가

```
"the", "is", "a" 같은 단어
→ 모든 문서에 등장 → DF 높음 → IDF 낮음 → TF-IDF 낮음
→ 중요하지 않은 단어 자동으로 걸러짐

"project", "management" 같은 단어
→ 특정 문서에만 등장 → DF 낮음 → IDF 높음 → TF-IDF 높음
→ 그 문서의 핵심 단어로 인식
```

---

## 코드

```python
from sklearn.feature_extraction.text import TfidfVectorizer

corpus = [
    "project planning and management",
    "budget control and risk management",
    "overseas business coordination"
]

vectorizer = TfidfVectorizer()
matrix = vectorizer.fit_transform(corpus)
# matrix 모양: (문서 수, 단어 수)
# 각 칸의 값 = 해당 문서에서 해당 단어의 TF-IDF 점수

print(matrix.shape)                          # (3, 단어수)
print(vectorizer.get_feature_names_out())    # 단어 목록 확인
print(matrix.toarray())                      # 행렬을 숫자로 출력
```

---

## 주요 파라미터

```python
TfidfVectorizer(
    max_features=1000,   # 상위 1000개 단어만 사용
    stop_words='english', # 불용어(the, is 등) 자동 제거
    ngram_range=(1, 2),  # 단어 1개 + 2개 조합도 사용
    min_df=2,            # 최소 2개 문서에서 등장한 단어만
)
```

---
