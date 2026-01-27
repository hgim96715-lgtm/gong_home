---
aliases:
  - 람다함수
  - Lambda
  - Map_Filter_Reduce
  - 익명함수
  - 고차함수
tags:
  - Python
related:
  - "[[RDD_Concept]]"
  - "[[Spark_Session_Deep_Dive]]"
  - "[[PySpark_Session_Context]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약

**"함수(Function)를 데이터(Data)에 마법처럼 한 방에 적용하는 기술."**

* **Lambda:** 이름 짓기 귀찮은 일회용 함수.
* **Map/Filter/Reduce:** 데이터 덩어리(List)를 **변환(Map), 거르기(Filter), 압축(Reduce)** 하는 3대장.

---
## 2. 문법 해부 (Syntax Anatomy) 

```python
# 문법: filter(함수, 데이터)
filter(lambda x: x % 2 == 0, data)
```

### ① `data`가 뭔가요? (The Iterable)

- **역할:** 함수가 처리해야 할 **재료 뭉치**입니다.
- 리스트(`[]`), 튜플(`()`), 셋(`{}`) 등 **순서대로 하나씩 꺼낼 수 있는(Iterable)** 모든 것이 올 수 있습니다.
- `filter`나 `map`은 이 `data`에서 하나씩 꺼내서 앞의 `lambda` 함수에 집어넣습니다.

### ② `list()`는 왜 씌우나요? (Lazy Evaluation) 

- **이유:** 파이썬 3부터는 `map`과 `filter`가 결과를 즉시 만들지 않고, **"준비만 된 상태(Iterator)"** 를 반환합니다. (메모리 절약을 위해!)
- **해결:** "결과를 눈으로 지금 당장 확인하고 싶어!" 라고 할 때 **`list()`** 로 감싸서 강제로 리스트로 변환해야 합니다.

```python
# 예시
nums = [1, 2, 3, 4]
result = map(lambda x: x * 2, nums)

print(result)       # 출력: <map object at 0x...> (준비 상태, 눈에 안 보임)
print(list(result)) # 출력: [2, 4, 6, 8] (리스트로 변환해야 보임)
```

---
## 실전 3대장 (Map, Filter, Reduce)

### ① Map (변신) 

- **의미:** "데이터 하나하나를 **변환**시켜라." (개수 변화 없음)
- **공식:** `{python}map(변환함수, 데이터)`
- **비유:** 흰색 자동차들이 공장에 들어가서 빨간색으로 도색되어 나옴.

```python
prices = [100, 200, 300]

# 모든 가격에 10% 수수료 붙이기
# list()를 안 쓰면 결과가 안 보임!
new_prices = list(map(lambda x: x * 1.1, prices))
# 결과: [110.0, 220.0, 330.0]
```

### ② Filter (검문) 

- **의미:** "조건이 **True(참)** 인 녀석만 통과시켜라." (개수 줄어듦)
- **공식:** `{python}filter(조건함수, 데이터)`
- **비유:** 공항 보안 검색대 (위험한 물건은 걸러냄).

```python
ages = [5, 12, 17, 20, 25]

# 성인(20세 이상)만 통과
adults = list(filter(lambda x: x >= 20, ages))
# 결과: [20, 25]
```

### ③ Reduce (압축) 

- **의미:** "데이터를 두 개씩 씹어먹으며 **하나로 합쳐라**." (결과는 딱 1개)
- **공식:** `{python}reduce(집계함수, 데이터)`
- **주의:** 파이썬 내장 함수가 아니라서 `import`가 필요함.

```python
from functools import reduce

nums = [1, 2, 3, 4, 5]

# 누적 합계 구하기 (x=누적값, y=새로운값)
# 1+2 -> 3
# 3+3 -> 6
# 6+4 -> 10
# 10+5 -> 15
total = reduce(lambda x, y: x + y, nums)
# 결과: 15 (리스트 아님, 숫자 하나)
```

---
##  데이터 엔지니어에게 왜 중요한가? (Spark 연결고리)

이 문법은 **Apache Spark**의 동작 방식과 100% 일치합니다.

|**Python (Local)**|**Spark (Distributed)**|**설명**|
|---|---|---|
|`map(...)`|`rdd.map(...)`|데이터를 변환함|
|`filter(...)`|`rdd.filter(...)`|데이터를 걸러냄|
|`reduce(...)`|`rdd.reduce(...)`|데이터를 하나로 합침|
|**`list(...)`**|**`rdd.collect()`**|**게으른 연산 결과를 강제로 가져옴**|

**깨달음:** 
- 파이썬에서 `list()`를 안 씌우면 결과가 안 보이는 것처럼, 스파크에서도 `collect()`(액션)를 안 하면 아무 일도 일어나지 않습니다. 
- 이것이 바로 **Lazy Evaluation(지연 연산)** 의 원리입니다!