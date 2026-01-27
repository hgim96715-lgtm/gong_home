---
aliases:
  - Transformation
  - Action
  - Lazy_Evaluation
  - parallelize
  - takeSample
  - 지연연산
tags:
  - Spark
  - RDD
related:
  - "[[RDD_Concept]]"
  - "[[Python_Lambda_Map]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Python_Random_Seed]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Iterating_Data]]"
  - "[[Python_Control_Flow]]"
  - "[[Spark_Data_Aggregation]]"
---
##  개념 한 줄 요약

**"스파크는 '계획(Transformation)'만 세우다가, '결과 내놔(Action)'라고 소리쳐야 비로소 움직인다."**

* **Transformation (대기조):** 데이터를 변형하는 **설계도**만 그립니다. (실행 X)
* **Action (활동조):** 실제 계산을 수행하고 **결과**를 가져옵니다. (실행 O)

---
##  `parallelize`가 뭔가요? (데이터 주입)

```python
# sc.parallelize(리스트)
rdd = sc.parallelize(range(1000))
```

- **의미:** "내 컴퓨터(Driver)에 있는 파이썬 리스트를 **클러스터(Workers)로 뿌려라(Distribute).**"

- **작동:**
    1. Driver가 `range(1000)` 데이터를 쪼갭니다(Partitioning).
    2. 여러 대의 Executor들에게 나눠줍니다.
    3. 이제부터 이 데이터는 **RDD(분산 데이터)** 가 됩니다.

- **분류:** 이것도 **Transformation(대기조)** 입니다. 실제로 데이터를 쫙 뿌리는 건 Action이 나올 때 합니다.

---
## 대기조 vs 활동조 (Transformation vs Action)

스파크 함수는 딱 두 종류로 나뉩니다. 이걸 구분 못하면 스파크를 못 씁니다.

### ① Transformation (변환 / 대기조)

- **특징:** 결과값으로 **새로운 RDD**를 반환합니다.
- **행동:** 계산하지 않습니다. "아, 나중에 이거 해야지" 하고 **족보(Lineage/DAG)** 에 적어두기만 합니다.

**대표 함수:**

- `map()`: 변신시키기
- `filter()`: 거르기
- `parallelize()`: 데이터 만들기
- `distinct()`: 중복 제거하기

### ② Action (행동 / 활동조) 

- **특징:** 결과값으로 **데이터(리스트, 숫자)** 를 반환하거나 파일을 저장합니다.
- **행동:** 드디어 움직입니다! 적어둔 족보를 보고 최적의 경로를 짜서 **실제로 계산**합니다.

**대표 함수:**

- `collect()`: 모든 데이터를 Driver로 가져와! (메모리 주의 )
- `count()`: 몇 개인지 세어봐!
- `countByValue()`: 값별로 몇 개인지 세어줘! (딕셔너리 반환)
- `take(n)`: 앞에서 n개만 가져와!
- `first()`: 맨 앞에 있는 거 딱 1개만 가져와! (데이터 맛보기용)
- `saveAsTextFile()`: 파일로 저장해!
- `foreach()`: 각 데이터마다 함수를 실행해! (출력, DB 저장 등)
- **`takeSample()`**: 랜덤으로 뽑아와!

---
### `takeSample(False, 5)` 분석

`{python}문법 :(중복허용여부, 가져올 데이터수 , 난수_기준값(seed))`

```python
# 문법: rdd.takeSample(withReplacement, num, seed=None)
rdd.takeSample(False, 5)
```

**`False` (withReplacement)**:
- **의미:** "한 번 뽑은 공을 다시 주머니에 넣을래?"
- `False`: **비복원 추출**. (한 번 뽑으면 끝. 중복 없음)
- `True`: **복원 추출**. (뽑고 다시 넣음. 같은 게 또 나올 수 있음)

**`5` (num)**:
- **의미:** "5개만 뽑아줘."

**`seed` (생략됨)**:
- **의미:** 랜덤 시드값. (이걸 고정하면 실행할 때마다 똑같은 랜덤값이 나옴)
