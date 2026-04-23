---
aliases:
  - numpy
  - array
  - ndarray
tags:
  - Numpy
related:
  - "[[00_Numpy_HomePage]]"
  - "[[CS_Vector_Matrix]]"
  - "[[Numpy_Array_creation]]"
  - "[[Numpy_Reshape]]"
  - "[[Numpy_Sparse_Matrix]]"
---


# Numpy_Array_Basics

---

## 1. NumPy가 뭔가

```
파이썬 기본 리스트는 느리고 수학 연산이 불편함
NumPy = 빠른 수치 연산을 위한 라이브러리

데이터 엔지니어링에서:
Pandas  → NumPy 위에서 동작
sklearn → NumPy 배열을 입출력으로 사용
Spark   → NumPy와 유사한 배열 개념 사용
```

---

## 2. ndarray란

**NumPy의 핵심 자료구조 = N차원 배열(N-Dimensional Array)**

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5])
print(type(arr))   # <class 'numpy.ndarray'>
print(arr)         # [1 2 3 4 5]
```

---

## 3. List vs ndarray

|비교 항목|파이썬 List|NumPy ndarray|
|---|---|---|
|속도|느림|빠름 (C언어 기반)|
|메모리|각 요소가 따로 저장|연속된 메모리에 저장|
|연산|for문 필요|벡터화 연산 가능|
|자료형|섞일 수 있음|하나의 dtype만 가능|
|다차원|리스트 안에 리스트|진짜 행렬 구조|

```python
# List 연산 → for문 필요
python_list = [1, 2, 3, 4, 5]
result = [x * 2 for x in python_list]   # [2, 4, 6, 8, 10]

# ndarray 연산 → 한 줄 (벡터화)
arr = np.array([1, 2, 3, 4, 5])
result = arr * 2                          # [2, 4, 6, 8, 10]
```

---

## 4. 연속 메모리 (왜 빠른가)

```
파이썬 List:
[1] → 메모리 주소 A
[2] → 메모리 주소 Z  (흩어져 있음)
[3] → 메모리 주소 M

→ 각 요소를 찾을 때마다 주소를 따라가야 함 → 느림


NumPy ndarray:
[1][2][3][4][5] → 메모리에 연속으로 저장

→ 한 번에 쭉 읽을 수 있음 → 빠름
→ CPU 캐시 효율도 좋음
```

---

## 5. dtype (데이터 타입)

ndarray는 **모든 요소가 같은 타입**이어야 해요.

```python
arr_int   = np.array([1, 2, 3])
arr_float = np.array([1.0, 2.0, 3.0])
arr_str   = np.array(["a", "b", "c"])

print(arr_int.dtype)    # int64
print(arr_float.dtype)  # float64
print(arr_str.dtype)    # <U1 (유니코드 문자)

# 타입 강제 지정
arr = np.array([1, 2, 3], dtype=float)
print(arr)       # [1. 2. 3.]
print(arr.dtype) # float64
```

```
⚠️ 정수 리스트에 실수 하나 섞으면?
np.array([1, 2, 3.5]) → dtype: float64 (자동으로 float으로 통일)
```

---

## 6. shape (배열의 구조)

```python
# 1차원
arr1 = np.array([1, 2, 3])
print(arr1.shape)  # (3,)  ← 요소 3개짜리 1차원

# 2차원
arr2 = np.array([[1, 2, 3],
                 [4, 5, 6]])
print(arr2.shape)  # (2, 3)  ← 2행 3열

# 3차원
arr3 = np.array([[[1, 2], [3, 4]],
                 [[5, 6], [7, 8]]])
print(arr3.shape)  # (2, 2, 2)  ← 2 × 2 × 2
```

```
shape 읽는 법:
(3,)    → 1차원, 요소 3개
(2, 3)  → 2차원, 2행 3열
(2,2,2) → 3차원, 2×2×2
```

---

## 7. 벡터화 (Vectorization)

**for문 없이 배열 전체에 한번에 연산 적용**

```python
arr = np.array([1, 2, 3, 4, 5])

# 전체에 한번에 적용
print(arr + 10)      # [11, 12, 13, 14, 15]
print(arr * 2)       # [2,  4,  6,  8,  10]
print(arr ** 2)      # [1,  4,  9,  16, 25]
print(arr > 3)       # [F,  F,  F,  T,  T]  ← Boolean

# 두 배열끼리 연산 (같은 shape)
a = np.array([1, 2, 3])
b = np.array([10, 20, 30])
print(a + b)         # [11, 22, 33]
```

---

## 8. 주요 속성 요약

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])

arr.shape    # (2, 3)     ← 형태
arr.ndim     # 2          ← 차원 수
arr.size     # 6          ← 전체 요소 수
arr.dtype    # int64      ← 데이터 타입
arr.itemsize # 8          ← 요소 하나의 바이트 크기
```

---
