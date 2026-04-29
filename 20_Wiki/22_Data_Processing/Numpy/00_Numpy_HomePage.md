
> NumPy 정복기!

---

## Core Concepts (핵심 기초)

```
파이썬 기본 List와의 차이점을 이해하고, 배열을 생성하는 기초 공사
```

|노트|핵심 개념|
|---|---|
|[[Numpy_Array_Basics]]|`ndarray` / `dtype` / `shape` / `List vs ndarray` / 연속 메모리 / 벡터화|
|[[Numpy_Array_creation]]|`np.array` / `np.zeros` / `np.ones` / `np.arange` / `np.random` / `np.eye` / `tolist` / `linspace`|

---

## Data Extraction (데이터 탐색 및 추출)

```
원하는 데이터만 쏙쏙 골라내는 기술
```

|노트|핵심 개념|
|---|---|
|[[Numpy_Indexing_slicing]]|`[행, 열]` / `[start:end:step]` / 음수 인덱스 / 다차원 슬라이싱|
|[[Numpy_Boolean_Fancy_Indexing]]|Boolean Mask / 조건 필터링 / Fancy Indexing / `np.where` / 마스크 씌우기|

---

## Shape Manipulation (배열 형태 다루기)

```
데이터 파이프라인에서 가장 에러가 많이 나는 '차원(Shape)' 맞추기 연습
```

| 노트                     | 핵심 개념                                                          |
| ---------------------- | -------------------------------------------------------------- |
| [[Numpy_Reshape]]      | `reshape` / `flatten` / `ravel` / `-1 자동 계산` / `T 전치` / 차원 맞추기 |
| [[Numpy_Concat_Split]] | `concatenate` / `vstack` / `hstack` / `split` / `axis` / 차원 유지 |

---

## Operations (연산과 통계)

```
for문을 버리고 NumPy의 진정한 속도를 끌어내는 벡터화 연산
```

|노트|핵심 개념|
|---|---|
|[[Numpy_Broadcasting]]|Broadcasting / 차원 자동 확장 / Shape 호환 규칙 / `(3,1) vs (1,3)`|
|[[Numpy_Math_Stats_Axis]]|`sum` / `mean` / `std` / `max` / `min` / `axis=0 행방향` / `axis=1 열방향`|

---

## Advanced (심화)

```
sklearn, TF-IDF, 대용량 처리 등 실전에서 만나는 NumPy 심화 개념
```

|노트|핵심 개념|
|---|---|
|[[Numpy_Sparse_Matrix]]|희소 행렬 / 0이 많은 행렬을 효율적으로 저장 / `(행, 열)` 좌표 저장 / CSR 형식 / `stored elements` 읽기 / `.toarray()`|

> 언제 나오나: TF-IDF 행렬 출력 시 `<Compressed Sparse Row sparse matrix>` 가 뜰 때 관련 실전 → [[Sklearn_TF_IDF]], [[Sklearn_Cosine_Similarity]]

---

## Trouble Shooting & Tips

```
자주 만나는 에러와 실수 모음
```

|상황|원인 & 해결|
|---|---|
|`shape` 불일치 에러|`matrix[0]` (1차원) 대신 `matrix[0:1]` (2차원) 사용|
|`.toarray()` 메모리 터짐|데이터 크면 Sparse 그대로 사용, 확인용으로만 쓰기|
|`dtype` 불일치 (`int` vs `float`)|`np.array([1,2,3], dtype=float)` 명시|

---

## 관련 노트

- [[CS_Vector_Matrix]] ← 벡터·행렬 개념 (NumPy 이해의 전제)
- [[Pandas]] ← NumPy 위에 만들어진 라이브러리
- [[Sklearn_TF_IDF]] ← Sparse Matrix가 실제로 만들어지는 곳