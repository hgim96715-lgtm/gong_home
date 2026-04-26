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
  - "[[Sklearn_CountVectorizer]]"
---
# Sklearn_TF_IDF — 텍스트 중요도 계산

## 한 줄 요약

```
텍스트를 숫자 벡터로 변환하는 방법
단순 빈도가 아니라 "이 단어가 이 문서에서만 중요한가" 를 반영
```

---

---

# ① TF-IDF 개념 ⭐️

## 용어 풀이

|용어|풀이|의미|
|---|---|---|
|TF|Term Frequency|하나의 문서에서 특정 단어가 등장하는 횟수|
|DF|Document Frequency|특정 단어가 등장하는 문서의 수|
|IDF|Inverse DF|DF 의 역수|
|TF-IDF|TF × IDF|최종 중요도 점수 (0 ~ 1)|

## IDF 가 왜 필요한가

```
"the", "is", "a" 같은 단어
→ 모든 문서에 등장 → DF 높음 → IDF 낮음 → TF-IDF 낮음
→ 중요하지 않은 단어 자동으로 걸러짐

"project", "management" 같은 단어
→ 특정 문서에만 등장 → DF 낮음 → IDF 높음 → TF-IDF 높음
→ 그 문서의 핵심 단어로 인식
```

## 계산 원리

```
TF(단어, 문서) = 해당 문서에서 단어 등장 횟수 / 문서 전체 단어 수

IDF(단어) = log(전체 문서 수 / 해당 단어 포함 문서 수)
            → 자주 나오는 단어 = IDF 낮음
            → 희귀한 단어 = IDF 높음

TF-IDF = TF × IDF
```

```
예시:
  문서 3개 / "management" 가 2개 문서에 등장

  IDF("management") = log(3 / 2) = 0.405

  문서1 에서 "management" 가 2번 / 전체 5단어
  TF = 2/5 = 0.4

  TF-IDF = 0.4 × 0.405 = 0.162
```

---

---

# ② 기본 사용법 ⭐️

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
print(vectorizer.get_feature_names_out())    # 단어 목록
print(matrix.toarray())                      # 행렬을 숫자로 출력
```

## 결과 해석

```python
import pandas as pd

# 보기 좋게 DataFrame 으로 변환
df = pd.DataFrame(
    matrix.toarray(),
    columns=vectorizer.get_feature_names_out()
)
print(df)

#    and  budget  business  control  coordination  management  overseas  planning  project  risk
# 0  0.0     0.0       0.0      0.0           0.0    0.377964       0.0  0.536429     0.536  0.0
# 1  0.0  0.447   ...
```

```
각 행 = 하나의 문서
각 열 = 단어 (vocabulary)
각 값 = TF-IDF 점수

0 에 가까울수록 = 해당 문서에서 중요하지 않음
1 에 가까울수록 = 해당 문서에서 중요한 단어
```

---

---

# ③ fit_transform vs transform ⭐️

```python
# fit_transform: 학습 + 변환 동시 (훈련 데이터에 사용)
matrix_train = vectorizer.fit_transform(train_corpus)

# transform: 이미 학습된 기준으로 변환만 (테스트/새 데이터)
matrix_test = vectorizer.transform(test_corpus)
```

```
fit_transform:
  어휘 사전(vocabulary) 만들기
  + 해당 문서들을 벡터로 변환

transform:
  이미 만들어진 어휘 사전 기준으로 변환만
  새 단어는 무시됨 (어휘 사전에 없으면 0)

⚠️ 테스트 데이터에 fit_transform 쓰면 안 됨
   훈련 기준으로 만든 사전과 달라져서 비교 불가
```

---

---

# ④ 주요 파라미터 ⭐️

```python
TfidfVectorizer(
    max_features=1000,     # 상위 N개 단어만 사용 (메모리 절약)
    stop_words='english',  # 불용어(the, is, a 등) 자동 제거
    ngram_range=(1, 2),    # 단어 조합 범위
    min_df=2,              # 최소 N개 문서에서 등장한 단어만
    max_df=0.9,            # 전체 90% 이상 문서에 나오면 제외
    sublinear_tf=True,     # TF 에 log 적용 (대용량 문서에 유용)
)
```

## ngram_range 상세 ⭐️

```
ngram_range = (최소, 최대)

(1, 1) = unigram만 (기본값)
  "project management" → ["project", "management"]

(2, 2) = bigram만
  "project management" → ["project management"]

(1, 2) = unigram + bigram ← 실무에서 가장 많이 씀
  "project management" → ["project", "management", "project management"]
```

```python
# unigram + bigram 예시
vectorizer = TfidfVectorizer(ngram_range=(1, 2))
matrix = vectorizer.fit_transform(corpus)

print(vectorizer.get_feature_names_out())
# ['and', 'budget', 'budget control', 'business', 'control',
#  'control and', 'management', 'overseas', 'planning', ...]
# ↑ 단어 1개짜리 + 2개짜리 조합 모두 포함
```

## bigram 이 왜 유용한가

```
"not good" → unigram: ["not", "good"]  → "good" 이란 단어로 분류될 위험
             bigram:  ["not good"]     → 부정 의미 보존 ✅

"project management" → 단어 조합이 하나의 의미
  unigram: "project" + "management" 따로
  bigram:  "project management" 를 하나의 단위로 처리

NCS ↔ O*NET 직무 매핑 같은 문맥:
  "데이터 엔지니어" 를 한 단위로 인식
  → 단순히 "데이터" + "엔지니어" 각각 처리보다 정확
```

## min_df / max_df

```python
# min_df: 너무 희귀한 단어 제거 (오타, 고유명사)
TfidfVectorizer(min_df=2)    # 최소 2개 문서에 등장해야 포함
TfidfVectorizer(min_df=0.01) # 전체 1% 이상 문서에 등장해야 포함

# max_df: 너무 흔한 단어 제거 (stop_words 보완)
TfidfVectorizer(max_df=0.9)  # 90% 이상 문서에 나오면 제외
TfidfVectorizer(max_df=0.8)  # 더 엄격하게 제거
```

---

---

# ⑤ vocabulary_ — 어휘 사전 확인

```python
vectorizer = TfidfVectorizer()
vectorizer.fit(corpus)

# 단어 → 인덱스 딕셔너리
print(vectorizer.vocabulary_)
# {'project': 5, 'planning': 4, 'management': 3, 'budget': 1, ...}

# 인덱스 → 단어 목록 (정렬된 배열)
print(vectorizer.get_feature_names_out())
# ['and', 'budget', 'business', 'control', ...]

# 어휘 크기
print(len(vectorizer.vocabulary_))
```

---

---

# ⑥ 희소 행렬 처리

```python
matrix = vectorizer.fit_transform(corpus)

# 기본 형태: 희소 행렬 (CSR 형식)
print(type(matrix))    # <class 'scipy.sparse._csr.csr_matrix'>
print(matrix.shape)    # (3, 단어수)

# 일반 배열로 변환 (소규모 데이터만)
dense = matrix.toarray()

# DataFrame 으로 변환 (분석용)
df = pd.DataFrame(dense, columns=vectorizer.get_feature_names_out())

# 희소 행렬 그대로 cosine_similarity 에 넘길 수 있음
from sklearn.metrics.pairwise import cosine_similarity
sim = cosine_similarity(matrix)
```

```
희소 행렬 (Sparse Matrix):
  대부분의 값이 0 인 행렬
  0 인 값은 저장하지 않음 → 메모리 절약
  toarray() 하면 일반 행렬로 변환 (메모리 많이 씀)

실무:
  문서 수천 개, 단어 수만 개 → 희소 행렬 필수
  소규모 테스트 → toarray() 로 눈으로 확인
```

---

---

# ⑦ 실전 패턴 — NCS ↔ O*NET 직무 매핑

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import pandas as pd

# 직무 설명 텍스트
ncs_jobs = ["데이터 수집 및 처리 파이프라인 구축"]
onet_jobs = [
    "Data Engineers build data pipelines",
    "Software developers create applications",
    "Database administrators manage systems"
]

# 합쳐서 한 번에 fit (같은 vocabulary 기준 적용)
all_texts = ncs_jobs + onet_jobs
vectorizer = TfidfVectorizer(ngram_range=(1, 2))
matrix = vectorizer.fit_transform(all_texts)

# NCS / O*NET 분리
ncs_vec  = matrix[:len(ncs_jobs)]
onet_vec = matrix[len(ncs_jobs):]

# 유사도 계산
sim = cosine_similarity(ncs_vec, onet_vec)
print(sim)
# [[0.42, 0.01, 0.03]]  ← 첫 번째 O*NET 직무와 가장 유사
```

---

---

# 전체 파라미터 한눈에

|파라미터|기본값|역할|
|---|---|---|
|`max_features`|None|상위 N개 단어만 사용|
|`stop_words`|None|불용어 제거 (`'english'`)|
|`ngram_range`|`(1,1)`|단어 조합 범위|
|`min_df`|1|최소 등장 문서 수|
|`max_df`|1.0|최대 등장 문서 비율|
|`sublinear_tf`|False|TF 에 log 적용|
|`analyzer`|`'word'`|분석 단위 (`'char'` 도 가능)|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|테스트 데이터에 fit_transform|어휘 사전이 달라짐|테스트엔 `transform` 만|
|`matrix.toarray()` 메모리 에러|대용량 희소 행렬|그대로 cosine_similarity 에 넘기기|
|ngram_range 효과 없음|단어가 너무 짧거나 min_df 조건|`min_df=1` + 코퍼스 확인|
|한글 처리 안 됨|기본 토크나이저가 영어 기준|`analyzer='char'` 또는 KoNLPy 연동|