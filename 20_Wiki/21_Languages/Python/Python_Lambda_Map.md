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
  - "[[Python_List_Comprehension]]"
---
# Python Lambda & Map/Filter/Reduce 완전 정복

## 개념 한 줄 요약

"함수(Function)를 데이터(Data)에 마법처럼 한 방에 적용하는 기술."

- **Lambda:** 이름 짓기 귀찮은 일회용 함수.
- **Map / Filter / Reduce:** 데이터 덩어리(List)를 **변환(Map), 거르기(Filter), 압축(Reduce)** 하는 3대장.

---
## Lambda(람다)의 정체: "일회용 함수"

거창하게 `def`로 정의해서 이름을 붙이기엔 너무 간단하고, 딱 한 번만 쓰고 버릴 함수가 필요할 때 사용한다.

---
## 가장 중요한 것: def vs lambda 문법 차이(헷갈림 주의!)

오류가 계속 나는 이유는 딱 하나다.
**`def`의 `return` 방식이랑 `lambda`의 방식이 다른데 섞어서 쓰고 있기 때문이다.**

```python
# ✅ def 방식  →  별도 줄에 return이 있다
def multiply(num1, num2):
    return num1 * num2


# ✅ lambda 방식  →  콜론(:) 뒤가 곧 return이다. return을 쓰지 않는다!
solution = lambda num1, num2: num1 * num2


# ❌ 자주 하는 실수들
lambda num1, num2 = num1 * num2   # SyntaxError! = 쓰면 안 됨
lambda num1: num2 = num1 * num2   # SyntaxError! num2는 입력 변수가 아님
lambda num1, num2: return num1 * num2  # SyntaxError! return 쓰면 안 됨
```

### 구조를 나란히 비교하면

```scss
def  함수이름 (입력변수)  :
         return  반환값

lambda   입력변수   :   반환값
```

>**핵심 암기:** `lambda` 에서는 콜론(`:`) 뒤에 오는 것이 **곧 return값**이다.
> `return` 키워드를 쓰는 순간 SyntaxError가 난다.

---
## 문법: `lambda 입력변수: 반환값`

```python
# 입력 변수가 1개일 때
square = lambda x: x ** 2
print(square(5))        # → 25

# 입력 변수가 2개일 때 (콤마로 구분)
multiply = lambda num1, num2: num1 * num2
print(multiply(3, 4))   # → 12

# 조건(삼항연산자)도 가능
check = lambda x: "양수" if x > 0 else "음수"
print(check(10))        # → 양수
```

---
## 핵심 실수: "정의" vs "실행" 헷갈리지 말기

람다는 정의만 한다고 실행되지 않는다. **반드시 호출(괄호)해야 한다.**

```python
# ❌ 틀린 예시 → 함수 덩어리만 리스트에 넣은 것 (계산 안 됨!)
answer = [lambda x: sum(x), [1, 2, 3]]
# 결과: [<function <lambda>...>, [1, 2, 3]]  ← 실행이 아니라 함수 객체가 들어감


# ✅ 맞는 방법 1: 변수에 담고 호출
my_func = lambda x: sum(x) ** 2
result = my_func([1, 2, 3])
# → 36


# ✅ 맞는 방법 2: 즉시 호출 → (함수)(재료) 형태로 괄호 두 번
result = (lambda x: sum(x) ** 2)([1, 2, 3])
#         ^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^
#         람다 정의              인자 넣어서 호출
# → 36


# ✅ 맞는 방법 3: map/filter에 태우기 (가장 자연스럽다)
result = list(map(lambda x: x + 10, [1, 2, 3]))
# → [11, 12, 13]
```

---
## list()는 왜 씌우나? (Lazy Evaluation)

파이썬 3부터는 `map`과 `filter`가 결과를 즉시 만들지 않고 **"준비만 된 상태(Iterator)"** 를 반환한다. (메모리 절약 때문)

```python
nums = [1, 2, 3, 4]
result = map(lambda x: x * 2, nums)

print(result)         # → <map object at 0x...>  눈에 안 보임!
print(list(result))   # → [2, 4, 6, 8]  list()로 감싸야 보임
```

---
## 실전 3대장 (Map, Filter, Reduce)

### ① Map — 변신 (개수 변화 없음)` map(함수, 리스트)`

**"데이터 하나하나를 변환시켜라."** 
흰색 자동차들이 공장에 들어가서 빨간색으로 도색되어 나오는 것.

```python
# 공식: map(함수, 리스트)

prices = [100, 200, 300]

new_prices = list(map(lambda x: x * 1.1, prices))
# → [110.0, 220.0, 330.0]
# 개수: 3개 → 3개 (그대로)
```

### ② Filter — 검문 (조건 True만 통과, 개수 줄어듦)`filter(조건함수, 리스트)`

**"조건이 True인 녀석만 통과시켜라."** 
공항 보안 검색대처럼 기준에 맞지 않으면 걸러낸다.

```python
# 공식: filter(조건함수, 리스트)

ages = [5, 12, 17, 20, 25]

adults = list(filter(lambda x: x >= 20, ages))
# → [20, 25]
# 개수: 5개 → 2개 (줄어듦)
```

### ③ Reduce — 압축 (결과가 딱 1개)`reduce(집계함수, 리스트)`

**"데이터를 두 개씩 씹어먹으며 하나로 합쳐라."** 파이썬 내장 함수가 아니라서 `import`가 필요하다.

```python
# 공식: reduce(집계함수, 리스트)
from functools import reduce

nums = [1, 2, 3, 4, 5]

total = reduce(lambda x, y: x + y, nums)
# 1+2 → 3
# 3+3 → 6
# 6+4 → 10
# 10+5 → 15
# → 15  (리스트 아님, 숫자 하나)
```


---
## 3대장 한눈에 비교

| 구분          | Map    | Filter           | Reduce            |
| ----------- | ------ | ---------------- | ----------------- |
| **역할**      | 변환     | 걸러내기             | 하나로 합치기           |
| **입력 개수**   | n개     | n개               | n개                |
| **출력 개수**   | **n개** | **n개 이하**        | **1개**            |
| **람다 반환값**  | 변환된 값  | **True / False** | 누적값               |
| **람다 인자 수** | 1개     | 1개               | **2개 (누적값, 현재값)** |
| **대표 사용 예** | 값 가공   | 조건 필터링           | 합계·곱·최댓값          |

---
## 데이터 엔지니어에게 왜 중요한가? (Spark 연결고리)

이 문법은 **Apache Spark**의 동작 방식과 100% 일치한다.

| 개념         | Python (Local)          | Spark (Distributed) | 설명                     |
| ---------- | ----------------------- | ------------------- | ---------------------- |
| **Map**    | `map(...)`              | `rdd.map(...)`      | 데이터를 **변환**함           |
| **Filter** | `filter(...)`           | `rdd.filter(...)`   | 조건에 맞는 데이터만 **걸러냄**    |
| **Reduce** | `functools.reduce(...)` | `rdd.reduce(...)`   | 여러 데이터를 **하나의 값으로 집계** |
| **결과 수집**  | `list(...)`             | `rdd.collect()`     | **게으른 연산 결과를 강제로 실행**  |

>파이썬에서 `list()`를 안 씌우면 결과가 안 보이는 것처럼, 스파크에서도 `collect()`를 안 하면 아무 일도 일어나지 않는다. 
>이것이 바로 **Lazy Evaluation(지연 연산)** 의 원리다.

---
## 최종 암기 요약

```python
# def는 return이 있다
def f(x):  return x * 2

# lambda는 콜론 뒤가 곧 return이다 (return 쓰지 마!)
f = lambda x: x * 2

# 입력 변수 여러 개 → 콤마로 구분
f = lambda x, y: x * y

# map/filter는 list()로 감싸야 결과가 보인다
list(map(lambda x: x * 2, [1,2,3]))     # → [2, 4, 6]
list(filter(lambda x: x > 1, [1,2,3]))  # → [2, 3]
```

---
## 📎 관련 노트

> **[[python_List_Comprehension]]** 참고
> 
> lambda + map 조합 대신 **한 줄로 더 깔끔하게 쓰고 싶을 때** 확인. 
> `map(lambda x: x*2, data)` 가 길게 느껴진다면 컴프리헨션으로 대체할 수 있다. 
> 특히 **if 조건 필터링**이나 **중첩 반복**이 필요한 상황에서 lambda보다 훨씬 직관적이다.

```python
# 이걸 lambda + map으로 쓰다가 복잡해지면
list(map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, data)))

# 컴프리헨션으로 바꾸면 한눈에 읽힌다
[x ** 2 for x in data if x % 2 == 0]
```

