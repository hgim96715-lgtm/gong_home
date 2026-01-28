---
aliases:
  - flatMap
  - map
  - filter
  - NarrowTransformation
  - sample
  - glom
tags:
  - Spark
related:
  - "[[Transformations_vs_Actions]]"
  - "[[Python_Lambda_Map]]"
  - "[[Python_String_Methods]]"
  - "[[00_Apache_Spark_HomePage]]"
---
##  개념 한 줄 요약 

**"데이터를 다른 모양으로 변신시키거나(`map`), 필요한 것만 남기거나(`filter`), 쪼개서 늘리는(`flatMap`) 함수들."**

* **공통점:** **Narrow Transformation (좁은 변환)** 입니다.
* **장점:** 다른 노드와 통신(Shuffle)할 필요 없이, **내 파티션 안에서 독립적으로** 수행되므로 엄청나게 빠릅니다.

---
## 1:1 변환의 정석: `map()` 

데이터 하나를 넣으면, **반드시 변환된 데이터 하나**가 나옵니다.
리스트의 개수가 변하지 않습니다. (Input 10개 -> Output 10개)

### ① 동작 원리

`func(x)`를 각 요소마다 실행해서 그 결과값을 그대로 자리에 넣습니다.

### ② 실전 예제 (CSV 파싱)

데이터가 `["홍길동,30", "임꺽정,40"]` 이렇게 문자열로 되어 있을 때, 튜플로 바꾸고 싶다면?

```python
data = ["홍길동,30", "임꺽정,40"]
rdd = sc.parallelize(data)

# x에는 "홍길동,30"이 들어옴 -> split으로 자름 -> (이름, 나이) 튜플로 만듦
# 리턴값: ("홍길동", 30)
def parse(line):
    name, age = line.split(",")
    return (name, int(age))

result = rdd.map(parse).collect()
print(result)
# 결과: [('홍길동', 30), ('임꺽정', 40)]
```

**핵심:** 입력이 문자열 1개였고, 출력도 튜플 1개입니다. **(1 대 1 매칭)**

---
## 1:N 뻥튀기: `flatMap()` 

데이터 하나를 넣으면, **여러 개(0개 ~ N개)** 가 튀어 나옵니다. 
결과물이 **납작하게(Flat)** 펴집니다.

### ① 동작 원리 (포장지 뜯기)

함수의 결과가 **리스트(List)** 일 때, 그 **리스트의 포장지를 뜯어서 내용물만** 꺼내놓습니다.
1. **Map을 썼을 때:** `[["A", "B"], ["C", "D"]]` (리스트 안에 리스트가 갇힘)
2. **FlatMap을 썼을 때:** `["A", "B", "C", "D"]` (리스트가 사라지고 알맹이만 남음)

### ② 실전 예제 (문장 쪼개기)

```python
lines = ["Hello World", "Spark is fast"]
rdd = sc.parallelize(lines)

# 상황: 공백 기준으로 잘라서 단어들을 나열하고 싶음.

# [비교 1] map을 썼을 때 (비극의 시작 😱)
mapped = rdd.map(lambda line: line.split(" "))
print(mapped.collect())
# 결과: [['Hello', 'World'], ['Spark', 'is', 'fast']]
# -> 2개의 덩어리로 나옴. 나중에 단어 세기 힘듦.

# [비교 2] flatMap을 썼을 때 (해피엔딩 🎉)
flat_mapped = rdd.flatMap(lambda line: line.split(" "))
print(flat_mapped.collect())
# 결과: ['Hello', 'World', 'Spark', 'is', 'fast']
# -> 5개의 단어로 쫙 펴짐. 이제 1씩 더해서 셀 수 있음!
```

---
## `map` vs `flatMap` 완벽 비교표

|**구분**|**map (맵)**|**flatMap (플랫맵)**|
|---|---|---|
|**비유**|**사과 껍질 깎기** 🍎|**콩깍지 까기** 🫛|
|**입력:출력 비율**|**1 : 1** (절대 변하지 않음)|**1 : N** (늘어나거나 줄어듦)|
|**함수 리턴값**|값 1개 (int, str, tuple...)|**리스트 (List, Iterable)**|
|**결과물 형태**|구조가 유지됨|**구조가 깨지고 1차원으로 펴짐**|
|**주 사용처**|데이터 형변환, 파싱, 계산|문장 쪼개기, 리스트 해체하기|

---
## 거름망: `filter()` 

조건이 **참(True)** 인 데이터만 남기고 나머지는 버립니다. 
데이터 개수가 줄어듭니다.

```python
nums = sc.parallelize([1, 2, 3, 4, 5])

# 짝수만 통과시켜라!
evens = nums.filter(lambda x: x % 2 == 0)

print(evens.collect()) 
# 결과: [2, 4]
```

**꿀팁:** 데이터를 읽자마자 불필요한 데이터(Null, 헤더, 이상한 값)는 `filter`로 최대한 빨리 제거해야, 뒤에 오는 연산들이 빨라집니다.

---
## 기타 유용한 변환들

### ① `sample(withReplacement, fraction)`

데이터의 일부를 **랜덤하게 추출**해서 새로운 RDD를 만듭니다. (Action인 `takeSample`과 다름!)

- **용도:** 머신러닝 테스트 데이터셋 만들 때.

```python
# 복원추출 X, 10%만 뽑기
sub_data = rdd.sample(False, 0.1)
```

### ② `glom()` (고급)

각 파티션에 있는 데이터를 **하나의 리스트로 묶어**줍니다.

- **용도:** "도대체 내 데이터가 파티션별로 어떻게 쪼개져 있나?" 눈으로 확인할 때 씁니다.

```python
rdd = sc.parallelize([1, 2, 3, 4], 2) # 파티션 2개

print(rdd.glom().collect())
# 결과: [[1, 2], [3, 4]] 
# -> 아하, 1번 파티션엔 1,2가 있고, 2번엔 3,4가 있구나!
```

---
### 요약

1. **일대일 교환**이다? -> **`map`**
2. **쪼개서 늘려야** 한다? 리스트 껍질을 벗겨야 한다? -> **`flatMap`**
3. **필요 없는 건 버려야** 한다? -> **`filter`**

"이 3가지는 스파크가 셔플 없이 가장 빠르게 처리하는 **'착한 함수들(Narrow)'** 이니까, 걱정 말고 마음껏 써도 좋아!"