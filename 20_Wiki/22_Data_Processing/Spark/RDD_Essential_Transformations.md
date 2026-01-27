---
aliases:
  - flatMap
  - map
  - reduceByKey
  - WordCount
  - 스파크단어세기
  - PairRDD
tags:
  - Spark
  - RDD
  - Syntax
  - Transformation
related:
  - "[[Transformations_vs_Actions]]"
  - "[[Python_Lambda_Map]]"
  - "[[Python_String_Methods]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 코드 해부 (Word Count Logic) 

사용자님이 가져온 코드는 **"문장을 쪼개서(Split) -> 단어마다 1점을 주고(Map) -> 같은 단어끼리 점수를 합치는(Reduce)"** 과정입니다.

```python
counts = text_file.flatMap(lambda line: line.split(" ")) \
             .map(lambda word: (word, 1)) \
             .reduceByKey(lambda a, b: a + b)
```

---
## 핵심 연산자 비교

### ① `map` vs `flatMap` (가장 많이 헷갈림!)

| **구분** | **map (1:1)**                       | **flatMap (1:N)**              |
| ------ | ----------------------------------- | ------------------------------ |
| **비유** | 사과 1개 껍질 깎기 -> 깎인 사과 1개             | 문장 1개 쪼개기 -> 단어 10개            |
| **입력** | `["Hello World"]`                   | `["Hello World"]`              |
| **결과** | `[["Hello", "World"]]` (리스트 안에 리스트) | `["Hello", "World"]` (납작하게 펴짐) |
| **특징** | 개수가 변하지 않음                          | **개수가 늘어남 (뻥튀기)**              |

**왜 `flatMap`을 썼나요?** 
- 줄(Line)은 하나지만, 그 안에 들어있는 단어(Word)는 여러 개니까요! 
- 리스트의 껍질을 벗겨서 단어들을 밖으로 꺼내야 합니다.

### ② `reduceByKey` (끼리끼리 뭉치기)

이 함수는 데이터가 **`(Key, Value)` 쌍**으로 되어 있을 때만 쓸 수 있습니다.

- **동작:** "Key가 같은 녀석들끼리 모여라! 그리고 Value를 가지고 연산(더하기)해라."
- **과정:**
	- 입력: `("Apple", 1)`, `("Banana", 1)`, `("Apple", 1)`
	- 셔플(Shuffle): 같은 Key끼리 같은 노드로 이동 
	- 연산: "Apple"은 1이 두 개네? -> `1 + 1 = 2`
	- 결과: `("Apple", 2)`, `("Banana", 1)`

---
## 전체 흐름 시뮬레이션

데이터가 `["Spark is cool", "Spark is fast"]` 두 줄이라고 가정해봅시다.

**Step 1. `flatMap(split)`**

- 줄 바꿈을 없애고 단어 단위로 쫙 폅니다.
- 결과: `["Spark", "is", "cool", "Spark", "is", "fast"]`

**Step 2. `map(word, 1)`**

- 단어마다 숫자 1을 붙여서 튜플을 만듭니다. (투표 용지 만들기)
- 결과: `[("Spark", 1), ("is", 1), ("cool", 1), ("Spark", 1), ("is", 1), ("fast", 1)]`


**Step 3. `reduceByKey(a + b)`**

- 같은 단어끼리 값을 더합니다.
- `("Spark", 1+1)`, `("is", 1+1)`, `("cool", 1)`, `("fast", 1)`
- 최종 결과: `[("Spark", 2), ("is", 2), ("cool", 1), ("fast", 1)]`

---
## 왜 `reduceByKey`가 중요한가? (vs `groupByKey`)

면접 단골 질문입니다.

- **`groupByKey`:** 일단 다 모아놓고 계산함. (데이터 이동량 많음 -> 느림)
- **`reduceByKey`:** **각자 자리(로컬)에서 미리 합치고(Combine)** 나서 모음. (데이터 이동량 적음 -> **빠름**)
- **결론:** 무조건 `reduceByKey`를 쓰는 게 성능에 좋습니다.