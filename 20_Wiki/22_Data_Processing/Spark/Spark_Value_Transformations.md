---
aliases:
  - mapValues
  - flatMapValues
  - values
  - countByValue
  - ValueTransformations
  - 파티션유지
tags:
  - Spark
  - RDD
related:
  - "[[Spark_Key_Transformations]]"
  - "[[Transformations_vs_Actions]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step4.ipynb
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step3.ipynb
---
##  개념 한 줄 요약 

**"키(Key)는 절대 건드리지 않고, 오직 값(Value)만 안전하게 변신시키는 함수들."**

* **핵심:** **셔플(Shuffle)이 없습니다.** (데이터 이동 X, 제자리에서 변신 ️)
* **장점:** `map`을 쓰면 파티션이 깨질 수 있지만, `mapValues`는 **파티션 구조를 그대로 유지**하기 때문에 훨씬 빠릅니다.

---
## 값만 바꾸기: `mapValues` (최고 중요 )

`(Key, Value)` 쌍에서 **Value만** 함수에 통과시킵니다. 
Key는 함수 안으로 들어오지도 않습니다.

### ① 기본 사용법 (Syntax)

```python
rdd = sc.parallelize([("Seoul", 1000), ("Busan", 2000)])

# "값(가격)에 10%를 더해라" (Key인 도시는 안 건드림)
new_rdd = rdd.mapValues(lambda price: price * 1.1)

# 결과: [("Seoul", 1100.0), ("Busan", 2200.0)]
```

### ② 실전 패턴: 평균 구하기 준비 운동 🏋️

`reduceByKey`에서 배웠던 **`(값, 1)` 만들기**를 할 때 `map`보다 `mapValues`를 쓰는 게 정석입니다.

```python
# [Bad] map 사용 (키도 같이 만지작거림 -> 스파크가 의심함 "혹시 키 바꿨어?")
# rdd.map(lambda x: (x[0], (x[1], 1)))

# [Good] mapValues 사용 (키는 쳐다도 안 봄 -> 스파크 안심 "파티션 유지!")
rdd.mapValues(lambda v: (v, 1))
```

---
## 값만 펼치기: `flatMapValues` 

**"키 하나에 값이 여러 개 뭉쳐 있는 경우"** 사용합니다. 
값의 껍질을 까서 **키와 함께 여러 줄로 복제(Explode)** 합니다.

**상황:** `("책제목", "저자1, 저자2, 저자3")` -> 저자별로 줄을 나누고 싶다.

```python
data = [("PythonBook", "Kim,Lee,Park")]
rdd = sc.parallelize(data)

# 1. 값을 쉼표로 쪼개서(split) 리스트로 만듦 -> ['Kim', 'Lee', 'Park']
# 2. flatMapValues가 리스트를 풀어서 키랑 짝지어줌
result = rdd.flatMapValues(lambda v: v.split(","))

# 결과: (키가 복제된 것을 보세요!)
# [
#   ("PythonBook", "Kim"), 
#   ("PythonBook", "Lee"), 
#   ("PythonBook", "Park")
# ]
```

---
## 값만 뽑기: `values()` 

키는 필요 없고, 값만 리스트로 챙기고 싶을 때 씁니다.

```python
rdd = sc.parallelize([("A", 10), ("B", 20)])

# 키("A", "B")는 버림
val_list = rdd.values().collect()

# 결과: [10, 20]
```

---
## 개수 세기: `countByValue()` ⚠️ [Action]

이 녀석은 `reduceByKey` 같은 변환(Transformation)이 아니라 **액션(Action)** 입니다! 
결과가 RDD가 아니라 **파이썬 딕셔너리**로 바로 나옵니다.

### ① 동작 방식

RDD 안에 있는 **데이터의 종류별 개수(빈도수)** 를 셉니다.

- **일반 RDD:** 각 아이템이 몇 개인지 셈.
- **Pair RDD:** `(Key, Value)` 쌍 자체가 몇 번 나왔는지 셈.

### ② 실전 예제 (로그 분석)

로그 레벨(INFO, ERROR)이 몇 번 떴는지 셀 때 최고입니다.

```python
log_rdd = sc.parallelize(["INFO", "ERROR", "INFO", "WARN", "INFO"])

# "각 단어가 몇 번 나왔니?"
result = log_rdd.countByValue()

print(result)
# 결과: {'INFO': 3, 'ERROR': 1, 'WARN': 1}
# (딕셔너리 형태이므로 result['INFO']로 접근 가능)
```

### 주의사항 (메모리 폭발)

`countByKey()`와 마찬가지로, 결과가 **Driver 메모리**로 한꺼번에 들어옵니다. 
데이터 종류(Unique Value)가 수십만 개라면 쓰면 안 됩니다. 
(그럴 땐 `map(x, 1).reduceByKey(...)` 패턴을 써야 안전합니다.)

---
## 요약: Key함수 vs Value함수

|**구분**|**Key 함수 (reduceByKey 등)**|**Value 함수 (mapValues 등)**|
|---|---|---|
|**대상**|Key가 같은 것들끼리|오직 Value 하나만|
|**셔플(Shuffle)**|**발생함 (데이터 이동 O)** 🚚|**발생 안 함 (이동 X)** ✅|
|**성능**|무거움 (네트워크 비용)|**가벼움 (메모리 연산)**|
|**용도**|집계, 정렬, 그룹핑|데이터 변환, 포맷 변경|

>"데이터 엔지니어링 튜닝의 제1원칙은 **'셔플(Shuffle)을 줄여라'** 야.
> 그래서 `(Key, Value)` 데이터를 다룰 때, 키를 바꿀 필요가 없다면 무조건 `map` 말고 **`mapValues`** 를 쓰는 습관을 들여야 해.
>  이 작은 습관이 나중에 대용량 데이터를 처리할 때 엄청난 속도 차이를 만든단다!"
