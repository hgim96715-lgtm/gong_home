---
aliases:
  - numpy
  - reshape
  - np.array
  - flatten
  - revel
  - Transpose
  - np.newaxis
  - np.expand_dims
  - squeeze
tags:
  - Numpy
related:
  - "[[00_Numpy_HomePage]]"
  - "[[Numpy_Array_Basics]]"
  - "[[CS_Vector_Matrix]]"
  - "[[Numpy_Concat_Split]]"
  - "[[Sklearn_Cosine_Similarity]]"
---

# Numpy_Reshape

> 전제 지식: [[Numpy_Array_Basics]], [[CS_Vector_Matrix]] 다음 단계: [[Numpy_Concat_Split]]

---

## 1. 왜 reshape이 필요한가

```
sklearn, TF-IDF, 딥러닝 등에서
"shape이 맞지 않습니다" 에러가 제일 많이 남

→ reshape은 데이터는 그대로, 형태(구조)만 바꾸는 것
```

---

## 2. reshape 기본

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5, 6])
print(arr.shape)  # (6,)  ← 1차원, 요소 6개

# 1차원 → 2차원 (2행 3열)
arr_2d = arr.reshape(2, 3)
print(arr_2d)
# [[1 2 3]
#  [4 5 6]]
print(arr_2d.shape)  # (2, 3)

# 1차원 → 2차원 (3행 2열)
arr_2d2 = arr.reshape(3, 2)
print(arr_2d2)
# [[1 2]
#  [3 4]
#  [5 6]]
```

```
⚠️ 핵심 규칙: reshape 전후 요소 수가 같아야 함
(6,) → (2, 3)  : 6개 = 2×3  ✅
(6,) → (2, 4)  : 6개 ≠ 2×4  ❌ 에러
```

---

## 3. -1 자동 계산

**한쪽 차원을 모를 때 -1을 넣으면 NumPy가 자동 계산**

```python
arr = np.array([1, 2, 3, 4, 5, 6])

arr.reshape(2, -1)   # 2행, 열은 자동 → (2, 3)
arr.reshape(-1, 3)   # 열은 3, 행은 자동 → (2, 3)
arr.reshape(-1, 1)   # 열은 1, 행은 자동 → (6, 1)
arr.reshape(1, -1)   # 행은 1, 열은 자동 → (1, 6)
```

```
실전에서 자주 쓰이는 패턴:
arr.reshape(-1, 1)  → 1열짜리 열벡터 (세로)
                       sklearn이 2차원을 요구할 때

arr.reshape(1, -1)  → 1행짜리 행벡터 (가로)
                       cosine_similarity(matrix[0:1]) 와 같은 효과
```

---

## 4. matrix[0] vs matrix[0:1] - reshape으로 이해

```python
matrix = np.array([[1, 2, 3],
                   [4, 5, 6],
                   [7, 8, 9]])

# 방법 1: 슬라이싱
print(matrix[0])    # [1 2 3]    shape: (3,)   ← 1차원
print(matrix[0:1])  # [[1 2 3]]  shape: (1, 3) ← 2차원

# 방법 2: reshape으로 같은 결과
print(matrix[0].reshape(1, -1))  # [[1 2 3]]  shape: (1, 3)

# cosine_similarity는 2차원 필요
# → matrix[0:1] 또는 matrix[0].reshape(1, -1) 둘 다 가능
```

---

## 5. flatten vs ravel

**2차원 → 1차원으로 펼치기**

```python
arr_2d = np.array([[1, 2, 3],
                   [4, 5, 6]])

# flatten: 복사본 반환 (원본 안 바뀜)
flat = arr_2d.flatten()
print(flat)         # [1 2 3 4 5 6]
flat[0] = 999       # 원본 arr_2d는 안 바뀜

# ravel: 가능하면 원본 공유 (메모리 효율)
rav = arr_2d.ravel()
print(rav)          # [1 2 3 4 5 6]
rav[0] = 999        # 원본 arr_2d 바뀔 수 있음

# 실무 팁:
# 원본 보존 필요 → flatten
# 속도/메모리 중요 → ravel
```

---

## 6. T 전치 (Transpose)

**행과 열을 뒤집기**

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])
print(arr.shape)    # (2, 3)

arr_T = arr.T
print(arr_T)
# [[1 4]
#  [2 5]
#  [3 6]]
print(arr_T.shape)  # (3, 2)  ← 행열 뒤집힘
```

```
언제 쓰나:
행렬 곱셈 전 shape 맞출 때
(2, 3) × (2, 3) → 에러
(2, 3) × (3, 2) → (2, 2) 가능  ← .T 사용
```

---

## 7. 차원 추가/제거

```python
arr = np.array([1, 2, 3])
print(arr.shape)  # (3,)

# 차원 추가 (np.newaxis)
arr_col = arr[:, np.newaxis]   # (3, 1) ← 열벡터
arr_row = arr[np.newaxis, :]   # (1, 3) ← 행벡터

# np.expand_dims (더 명시적)
arr_col = np.expand_dims(arr, axis=1)  # (3, 1)
arr_row = np.expand_dims(arr, axis=0)  # (1, 3)

# 차원 제거 (squeeze)
arr_back = arr_col.squeeze()   # (3,) ← 크기 1인 차원 제거
```

---

## 8. 자주 나오는 에러

```python
# ❌ 요소 수 불일치
arr = np.array([1, 2, 3, 4, 5, 6])
arr.reshape(2, 4)
# ValueError: cannot reshape array of size 6 into shape (2,4)
# 6 ≠ 2×4=8

# ✅ 해결
arr.reshape(2, 3)   # 6 = 2×3  OK
arr.reshape(3, 2)   # 6 = 3×2  OK
arr.reshape(-1, 2)  # 자동 계산 → (3, 2)

# ❌ 1차원을 cosine_similarity에 넣을 때
cosine_similarity(matrix[0], matrix[1:])
# ValueError: Expected 2D array

# ✅ 해결
cosine_similarity(matrix[0:1], matrix[1:])
cosine_similarity(matrix[0].reshape(1, -1), matrix[1:])
```

---

## 관련 노트

- [[Numpy_Array_Basics]] ← shape, ndim 개념 (선행)
- [[CS_Vector_Matrix]] ← 1차원/2차원 의미
- [[Numpy_Concat_Split]] ← 배열 합치고 쪼개기
- [[Sklearn_Cosine_Similarity]] ← matrix[0:1] reshape 실전 적용
- [[00_Numpy_HomePage]]