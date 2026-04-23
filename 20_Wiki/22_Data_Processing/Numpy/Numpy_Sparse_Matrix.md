---
aliases:
  - 희소 행령
  - Spare_Matrix
  - matrix[0]
  - matrix[1]
tags:
  - Numpy
related:
  - "[[00_Numpy_HomePage]]"
  - "[[CS_Vector_Matrix]]"
  - "[[Numpy_Array_Basics]]"
  - "[[Sklearn_TF_IDF]]"
  - "[[Sklearn_Cosine_Similarity]]"
---

# Sparse Matrix (희소 행렬)

---

## 1. 왜 Sparse Matrix가 필요한가

### 문제 상황

```
TF-IDF 행렬 크기:
문서 1017개 × 단어 4867개 = 4,949,739개 칸

그런데 한 문서에서 실제로 쓰는 단어는 40~459개
나머지 4400~4800개 칸은 전부 0

0을 전부 저장하면:
4,949,739개 × 8바이트 = 약 40MB 낭비
```

### 해결책: 0이 아닌 것만 저장

```
일반 행렬 (Dense):         Sparse Matrix:
┌                    ┐     (0, 4435) → 0.290
│0  0  0.29  0  0.64│     (0, 615)  → 0.648
│0  0  0     0  0   │     (0, 402)  → 0.363
│0  0.14 0   0  0   │     (1, 4435) → 0.142
└                    ┘     ...
모든 칸 저장               0 아닌 칸만 저장
```

---

## 2. 출력 읽는 법

```python
matrix = vectorizer.fit_transform(corpus)
print(matrix[0])
```

```
<Compressed Sparse Row sparse matrix of dtype 'float64'
  with 459 stored elements and shape (1, 4867)>

  Coords    Values
  (0, 4435) 0.29073
  (0, 615)  0.64856
  (0, 402)  0.36319
  ...
```

### 읽는 법

```
(0, 4435)  0.29073
 ↑    ↑       ↑
 행   열      값
번호  번호

행 번호: 이 출력은 matrix[0] 즉 1개 행이라 항상 0
열 번호: 단어 번호 (4435번째 단어)
값:      그 단어의 TF-IDF 점수

→ "4435번째 단어가 이 문서에서 TF-IDF 0.29의 중요도를 가짐"
```

### 어떤 단어인지 확인하는 법

```python
# 단어 번호 → 단어 이름 확인
vocab = vectorizer.get_feature_names_out()
print(vocab[4435])  # 4435번째 단어 이름 출력
print(vocab[615])   # 615번째 단어 이름 출력
```

---

## 3. Dense로 변환해서 보기

```python
# Sparse → 일반 numpy 배열로 변환
dense = matrix.toarray()
print(dense.shape)    # (1017, 4867)
print(dense[0][:10])  # 0번 문서의 처음 10개 단어 점수
# [0. 0. 0. 0. 0. 0. 0. 0. 0. 0.029]  ← 대부분 0
```

> ⚠️ 데이터가 크면 `.toarray()` 는 메모리 터질 수 있음 확인용으로만 쓰고 실제 계산은 Sparse 그대로 사용

---

## 4. matrix[0]과 matrix[1]이 다른 이유

```python
print(matrix[0])  # 459개 stored elements
print(matrix[1])  # 40개 stored elements
```

```
matrix[0] = NCS 번역 텍스트
→ 문장이 매우 길고 다양한 단어 포함
→ 459개 단어가 0이 아님

matrix[1] = O*NET 첫번째 직업 (Chief Executives)
→ 짧은 설명 한 단락
→ 40개 단어만 0이 아님

둘 다 4867차원 벡터지만
실제로 값이 있는 위치(단어)가 다름
→ 겹치는 단어 위치의 값을 곱해서 더하면 유사도
```

---

## 5. CSR이 뭔지 (참고)

출력에서 `Compressed Sparse Row` 라고 나왔는데:

```
Row(행) 기준으로 압축 저장한다는 의미
sklearn의 TfidfVectorizer가 기본으로 CSR 형식 사용
cosine_similarity도 CSR을 그대로 받을 수 있음
→ 직접 변환할 필요 없음
```

---

## 6. 정리

```
Sparse Matrix = 0이 많은 행렬을 효율적으로 저장하는 방법
               (행, 열) 좌표 + 값 형태로 0이 아닌 것만 저장

출력에서 (0, 4435) 0.29 → "0번 행, 4435번 단어, 점수 0.29"

.toarray() → 일반 배열로 변환 (확인용)
.shape     → (문서 수, 단어 수)
```

---
