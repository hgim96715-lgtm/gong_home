---
aliases:
  - Aggregation
  - 집계
  - countByValue
  - 빈도수
  - 히스토그램
  - Spark정렬
tags:
  - Spark
  - Python
  - DataAnalysis
  - Pattern
related:
  - "[[RDD_Essential_Transformations]]"
  - "[[Transformations_vs_Actions]]"
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Dictionaries]]"
---
## 1개념 한 줄 요약

**"데이터를 그룹별로 뭉쳐서 개수를 세고(`countByValue`), 그 결과를 내 입맛대로 줄 세우는(`sorted`) 기술."**

* **목표:** "A등급은 몇 명이고, B등급은 몇 명이지?" (분포 확인)

---
##  핵심 코드 패턴 (Code Pattern) 

이 3단 콤보는 현업에서 로그 분석할 때 매일 씁니다.

```python
# 1. 데이터 추출 (Transformation) - "Tom A" 에서 "A"만 뽑기
grade = text_file.map(lambda line: line.split(" ")[1])

# 2. 개수 세기 (Action) - "A가 몇 개?"
grade_count = grade.countByValue() 
# 결과(Dictionary): {'A': 30, 'B': 15, 'C': 5}

# 3. 결과 정렬 (Pure Python) - "개수 많은 순서로 보여줘"
for grade, count in sorted(grade_count.items(), key=lambda item: item[1], reverse=True):
    print(f"{grade}: {count}")
```

---
## 상세 분석 (Deep Dive)

### ① `map(...[1])` 의 비밀

```python
lambda line: line.split(" ")[1]
```

- **동작 원리:**
    1. `line` ("Tom A")을 공백으로 쪼갭니다 -> `['Tom', 'A']` (리스트)
    2. 뒤에 `[1]`을 붙여서 **1번 인덱스(두 번째 칸)**를 가져옵니다. -> `'A'`

- **⚠️ 주의:** 빈 줄이나 공백이 없는 줄이 들어오면 `IndexError`로 뻗어버립니다. 반드시 `filter()`로 안전장치를 걸어야 합니다.

```python
grade = text_file.filter(lambda line: len(line) > 0) \
		 .map(lambda line: line.split(" ")[1])
```

### ② `countByValue()` (자동 카운팅 기계)

- **역할:** RDD에 있는 값들의 종류별 개수를 세줍니다.
    
- **특징:**
    - `map` + `reduceByKey` + `collect`를 한 번에 해주는 **Action 함수**입니다.
    - 결과값은 스파크 RDD가 아니라 **파이썬 딕셔너리(`dict`)** 입니다.

**🚨 경고:**

- 결과가 **Driver(내 컴퓨터) 메모리**로 들어옵니다.
- 종류(Key)가 **수백만 개(예: 유저 ID)** 라면 메모리가 터질 수 있습니다.
- 성별, 등급, 지역코드처럼 **종류가 적은 경우**에만 써야 합니다.

### ③ 파이썬 정렬 공식 (`sorted`)

이건 스파크가 아니라 **파이썬 문법**입니다. 딕셔너리는 순서가 없어서 강제로 줄을 세워야 합니다.

```python
sorted(grade_count.items(), key=lambda item: item[1], reverse=True)
```

1. **`grade_count.items()`**: 딕셔너리를 튜플 리스트로 바꿉니다. `[('A', 30), ('C', 5), ('B', 15)]`
2. **`key=lambda item: item[1]`**: 정렬의 기준(Key)을 정합니다.
	- `item[0]`은 등급 이름('A', 'B'...)
	- **`item[1]`은 개수(30, 15...)** -> "개수를 기준으로 정렬해라!"
3. **`reverse=True`**: 내림차순 (큰 수가 먼저 오게)

---
## `countByValue` vs `reduceByKey`

|**구분**|**countByValue()**|**reduceByKey()**|
|---|---|---|
|**종류**|**Action** (결과를 당장 내놔)|**Transformation** (계획만 세워놔)|
|**결과물**|파이썬 딕셔너리 (내 컴퓨터 메모리)|분산된 RDD (여러 서버에 흩어짐)|
|**용도**|카테고리가 적을 때 (성별, 요일)|카테고리가 엄청 많을 때 (단어, 유저ID)|
|**속도**|코드가 짧고 간편함|대용량 처리에 훨씬 안정적임|


---
### 팁

"이 코드는 **데이터 품질 검사(QA)** 할 때 진짜 많이 써.
데이터를 딱 받자마자 `countByValue()` 한번 돌려보는 거야.
만약 성별 컬럼에 `{'M': 100, 'F': 90, 'Alien': 1}` 처럼 이상한 값이 섞여 있으면 바로 찾아낼 수 있거든.
**데이터의 분포를 눈으로 확인하고 싶을 때** 가장 먼저 꺼내야 할 도구야!"