---
aliases:
  - reduceByKey
  - groupByKey
  - sortByKey
  - countByKey
  - KeyTransformations
  - 셔플
tags:
  - Spark
related:
  - "[[Spark_General_Transformations]]"
  - "[[Transformations_vs_Actions]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약 

**"키(Key)가 같은 녀석들끼리 모으거나, 키를 기준으로 줄을 세우는 함수들."**

* **핵심:** **셔플(Shuffle)** 이 발생합니다. (데이터가 네트워크를 타고 다른 노드로 이동함 )
* **목표:** 흩어져 있는 데이터를 키(Key) 기준으로 정리정돈하기.

---
## 합치기 대장: `reduceByKey` (끼리끼리 뭉치기) 

`(Key, Value)` 쌍으로 된 데이터에서 **같은 Key를 가진 Value들끼리** 합치는 함수입니다.
가장 많이 쓰고, 가장 성능이 좋습니다.

* **전제조건:** 데이터가 반드시 `(Key, Value)` 튜플 형태여야 함.
* **동작:** **"셔플(Shuffle)"** 을 통해 같은 키를 가진 녀석들을 한곳으로 모은 뒤 연산합니다.

### ① `a`와 `b`의 정체 (눈덩이 이론 ☃️)

`{python}reduceByKey(lambda a, b: ...)` 에서 두 변수의 역할이 다릅니다.

* **`a` (Accumulator):** 지금까지 굴려서 커진 **눈덩이 (누적값)**
* **`b` (Current):** 길바닥에 놓인 **새로운 눈 (현재값)**

### ② 기본: 숫자 하나 더하기

```python
# 같은 키끼리 값 더하기 (Word Count)
rdd.reduceByKey(lambda a, b: a + b)
```

상황: **'Seoul'** 이라는 키에 데이터가 3개(`[10, 20, 30]`) 들어왔을 때

|**단계**|**a (눈덩이)**|**b (새로운 눈)**|**연산 (a + b)**|**결과 (새로운 a)**|
|---|---|---|---|---|
|**1단계**|`10` (첫 번째 값)|`20` (두 번째 값)|10 + 20|**30**|
|**2단계**|**30** (아까 만든 거)|`30` (세 번째 값)|30 + 30|**60**|
|**끝**|**60**|없음|-|최종 리턴|


### ③ [심화] 튜플끼리 더하기 (평균 구할 때 필수!) 

값 하나가 아니라 **`(총합, 개수)`** 처럼 튜플 뭉치로 되어 있을 때는, **인덱스(자리)** 를 맞춰서 끼리끼리 더해야 합니다.

```python
# 데이터: ("Seoul", (1000, 1)), ("Seoul", (2000, 1))
# 목표: 금액은 금액끼리[0], 개수는 개수끼리[1] 더하고 싶다!

rdd.reduceByKey(lambda a, b: (a[0] + b[0], a[1] + b[1]))
```

- **`a` (눈덩이):** `(1000, 1)`
- **`b` (새 눈):** `(2000, 1)`
- **연산:**
    - `a[0] + b[0]` -> `1000 + 2000` = **3000** (총합)
    - `a[1] + b[1]` -> `1 + 1` = **2** (개수)
- **결과:** `(3000, 2)` -> 이 튜플이 다시 `a`가 되어 다음 눈덩이가 됩니다.

---
## 그냥 모으기: `groupByKey` (함부로 쓰지 마세요 )

`reduceByKey`와 다르게 **연산(더하기)을 하지 않고, 그냥 리스트로 묶어서** 가져옵니다.

### ① 동작 방식 (쓰레기봉투 이론)

- **`reduceByKey`:** 각자 집에서 쓰레기를 **압축(Combine)** 해서 내놓음. (가볍다)
- **`groupByKey`:** 쓰레기를 **그냥 봉투째로** 트럭에 실어 보냄. (무겁다 -> 네트워크 터짐)

### ② 결과 모양 확인하기

리스트가 아니라 `ResultIterable`이라는 반복 가능한 객체로 반환됩니다.

```python
rdd = sc.parallelize([("A", 1), ("A", 2), ("B", 1)])

grouped = rdd.groupByKey()
result = grouped.collect()

# 결과: [('A', <pyspark.resultiterable.ResultIterable object>), ('B', ...)]
# 눈으로 보려면 list()로 변환해야 함
# [('A', [1, 2]), ('B', [1])]
```

### ③ 언제 쓰나요?

- 합계나 평균을 구하는 게 아니라, **진짜로 데이터를 묶어서 보관해야 할 때.**
- 예: `("유저ID", "접속로그")` -> 유저별 접속 로그를 시간순으로 나열해서 보고 싶을 때.

---
## 줄 세우기: `sortByKey` 

키(Key)를 기준으로 데이터를 정렬합니다.

### ① 기본 사용법 (오름차순/내림차순)

```python
# 데이터: (이름, 점수)
pairs = sc.parallelize([("B_Team", 10), ("A_Team", 20), ("C_Team", 5)])

# 1. 오름차순 (기본값, True) -> A, B, C 순서
sorted_up = pairs.sortByKey(ascending=True)
# 결과: [("A_Team", 20), ("B_Team", 10), ("C_Team", 5)]

# 2. 내림차순 (False) -> C, B, A 순서
sorted_down = pairs.sortByKey(ascending=False)
# 결과: [("C_Team", 5), ("B_Team", 10), ("A_Team", 20)]
```

### ② 주의사항 (타입 일치)

키의 타입이 섞여 있으면 에러가 납니다. (숫자는 숫자끼리, 문자는 문자끼리!)

- `Key`가 문자열(`String`)이면 사전 순서 (가나다, ABC)
- `Key`가 숫자(`Int`)면 크기 순서 (1, 2, 3)

---
## 키 관련 유틸리티 (Action 주의!)

### ① `keys()`

값(Value)은 버리고, **"어떤 키들이 있는가?"** 만 리스트로 뽑아냅니다.

```python
# [("A", 1), ("B", 2)] -> ["A", "B"]
key_list = pairs.keys().collect()
```

### ② `countByKey()` ⚠️ [Action]

이 녀석은 변환(Transformation)이 아니라 **액션(Action)** 입니다! 
결과가 RDD가 아니라 파이썬 **딕셔너리(Dictionary)** 로 드라이버에 바로 들어옵니다.

- **동작:** 키별로 데이터가 몇 개인지 셉니다.
- **주의:** 키 종류가 수억 개라면? 드라이버 메모리가 터집니다. (OOM)

```python
data = [("A", 10), ("B", 20), ("A", 30)]
rdd = sc.parallelize(data)

# RDD가 아님! 그냥 파이썬 딕셔너리임.
result = rdd.countByKey() 

print(result)
# 결과: {'A': 2, 'B': 1} (A는 2번, B는 1번 등장)
# 접근: result['A'] -> 2
```

---
### 요약

1. **합치고 싶다?** -> 무조건 **`reduceByKey`** (성능 최강, 눈덩이 이론 기억하기 ☃️)
2. **묶어서 리스트로 만들어야 한다?** -> 어쩔 수 없이 **`groupByKey`** (메모리 조심)
3. **랭킹을 매겨야 한다?** -> **`sortByKey`**
4. **`countByKey`** 는 **Action**이라서 결과가 바로 튀어나온다! (메모리 주의)