---
aliases:
  - List
  - Tuple
  - 리스트
  - 튜플
  - Slicing
  - 슬라이싱
  - 배열
tags:
  - Python
related:
  - "[[Python_Control_Flow]]"
  - "[[]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Variables_Types]]"
  - "[[00_Python_HomePage]]"
---

##  개념 한 줄 요약 

데이터를 순서대로 담는 **기차** 같은 자료구조입니다.

| 구분 | **List (리스트) `[]`** | **Tuple (튜플) `()`** |
| :--- | :--- | :--- |
| **비유** | 화물 기차 (짐을 뺐다 꼈다 가능) | 밀봉된 기차 (출발하면 수정 불가) |
| **특징** | **Mutable (수정 가능)** | **Immutable (수정 불가)** |
| **용도** | 데이터 수집, 가공, 변환 | 설정값, 좌표, 함수 리턴값 |

---
## 왜 필요한가? (Why) 

### ① Airflow 활용

* **List:** 태스크의 실행 순서를 묶을 때 씁니다.
    * `[task_extract, task_transform] >> task_load`
* **Tuple:** 함수에서 **여러 개의 값**을 한 번에 반환할 때 씁니다.
    * `return ("success", 200)`

### ② 변수 지옥 탈출

* 데이터를 묶어서 관리하지 않으면 `file1`, `file2`, `file3`... 처럼 변수 이름을 100개 지어야 하는 지옥이 펼쳐집니다.

---
## List (리스트): 만능 상자 

가장 많이 쓰고, 기능도 가장 많습니다.
**(인덱스는 무조건 0부터 시작합니다!)**

### A. 만들고 조회하기

```python
# 1. 생성
tasks = ["extract", "transform", "load"]

# 2. 조회 (Indexing) - 0: 첫번째, 1: 두번째
print(tasks[0])  # 'extract'
print(tasks[-1]) # 'load' (맨 뒤에서 첫 번째)

# 3. 개수 세기 (len)
print(len(tasks)) # 3
```

### B. 수정하기 (필수 함수 4대장)

#### ① `append()`: 덩어리째 추가 (박스 채로 넣기)

리스트를 넣으면 **리스트 자체가 하나의 아이템**으로 들어갑니다.

```python
a = [1, 2]
b = [3, 4]
a.append(b)
print(a) # [1, 2, [3, 4]] <-- 헉! 리스트 안에 리스트가 들어감!
```

#### ② `extend()`: 내용물만 쏟아붓기 (낱개로 추가)

리스트의 **알맹이만 꺼내서** 합칩니다.

```python
a = [1, 2]
b = [3, 4]

# 주의: 반환값이 None임! (변수에 담으면 데이터 날아감)
# wrong = a.extend(b) <-- (X) wrong은 None이 됨.

a.extend(b) 
print(a) # [1, 2, 3, 4] <-- 깔끔하게 합쳐짐!
```

#### ③ `+` 연산자: 새로운 리스트 생성 

원본(`a`)을 건드리지 않고, **새로운 복사본**을 만듭니다.

```python
a = [1, 2]
b = [3, 4]

c = a + b 
print(c) # [1, 2, 3, 4] (새로운 리스트)
print(a) # [1, 2] (원본은 멀쩡함!)
```

### [비교] 합치는 방법 3대장 요약

|**방식**|**동작 방식**|**원본 변경**|**반환값 (Return)**|
|---|---|---|---|
|**`append(b)`**|`b`를 **통째로** 맨 뒤에 넣음|✅ **O (변경됨)**|`None` (없음)|
|**`extend(b)`**|`b`의 **알맹이만** 풀어서 넣음|✅ **O (변경됨)**|`None` (없음)|
|**`a + b`**|알맹이를 합쳐서 **새것**을 만듦|❌ **X (유지됨)**|**새 리스트 (`[..]`)**|

>**주의:** `mix = data.extend(nums)` 라고 쓰면 `mix`는 `None`이 됩니다! 
>`extend`는 **"원본(`data`)을 고쳤으니 그걸 갖다 쓰세요"** 라는 뜻입니다.
>보통
>데이터를 계속 수집할 땐 가벼운 **`append`** 나 **`extend`** 를 쓰고,
>원본 데이터를 보존해야 할 때만 **`+`** 를 씁니다.


### [심화] 삭제 기술: `pop` vs `del` vs `remove`

|**구분**|**pop()**|**del**|**remove()**|
|---|---|---|---|
|**방식**|함수 (`.pop()`)|키워드 (`del ...`)|함수 (`.remove()`)|
|**기능**|꺼내고 삭제|그냥 삭제|값으로 찾아서 삭제|
|**값 반환**|✅ **있음 (꺼낸 값)**|❌ **없음 (Void)**|❌ **없음 (None)**|
|**변수 저장**|**가능 (`a = list.pop()`)**|불가능|불가능|
|**용도**|"꺼내서 쓸 거야"|"그냥 지워버려"|"값(내용)만 알아"|

#### `pop()`: 꺼내서 변수에 담기 (뽑기 기계) 

데이터를 리스트에서 삭제하면서, 동시에 **"내가 가질 수 있게(Return)"** 돌려줍니다.

* **`pop()` (빈 괄호):** 맨 뒤에 있는 녀석을 뽑습니다. (기본값)
* **`pop(idx)` (숫자 지정):** 해당 **인덱스(위치)** 에 있는 녀석을 콕 집어 뽑습니다.

```python
data = ["A", "B", "C", "D"]

# ① 맨 뒤 뽑기 (기본)
last = data.pop() 
print(last) # "D" (꺼낸 값)
print(data) # ["A", "B", "C"] ("D"는 사라짐)

# ② 특정 위치 뽑기 (응용)
# 1번 인덱스("B")를 뽑아줘!
second = data.pop(1)
print(second) # "B" (꺼낸 값)
print(data)   # ["A", "C"] ("B"도 사라지고, "C"가 앞당겨짐)
```

>"**`pop(0)`** 을 하면 **맨 앞**에 있는 걸 꺼내오겠지? 이걸 **'큐(Queue)'** 처럼 쓴다고 해. 
>반대로 그냥 **`pop()`** 하면 **맨 뒤**에서 꺼내니까 **'스택(Stack)'** 이고. 
>단, 주의할 점! 맨 앞(`pop(0)`)을 빼면 뒤에 있는 100만 개 데이터가 전부 한 칸씩 앞으로 당겨와야 해서 **속도가 좀 느려.** 
>그래서 데이터가 엄청 많을 때는 `deque`라는 다른 도구를 쓰긴 하는데, 일단은 **`pop(인덱스)`로 원하는 위치를 쏙 뺄 수 있다**는 것만 알면 충분해!"


#### `del`: 가차 없이 삭제 (파쇄기)

값을 돌려주지 않고 그냥 메모리에서 날려버립니다. `pop`과 다르게 **반환값이 없습니다.**

```python
data = [1, 2, 3]

# 1번 인덱스를 그냥 지워! (변수에 저장 못 함)
del data[1]

print(data) # [1, 3]
```

#### `remove()`: 값으로 찾아서 삭제 

인덱스(순서)를 모를 때, 내용물(`Value`)로 찾아서 지웁니다.

```python
data = ["Red", "Green", "Blue"]

data.remove("Green") # "Green"을 찾아서 지워라
# data.remove("Black") <-- 없는 걸 지우려 하면 에러(ValueError) 남!
```

---
## Tuple (튜플): 안전 금고 

한 번 만들면 내용을 절대 바꿀 수 없습니다. 
그래서 **"안전해야 하는 설정값"** 에 씁니다.


### A. 생성과 조회

```python
# 소괄호 () 사용
db_info = ("localhost", 5432)

print(db_info[0]) # 'localhost'
```

### B.  수정 불가 (에러 발생)

```python
# db_info[0] = "127.0.0.1" 
# -> TypeError: 'tuple' object does not support item assignment
```

### C. Packing & Unpacking (포장 뜯기) 

파이썬만의 아주 편리한 기능입니다. 변수 여러 개에 값을 한 번에 담습니다.

```python
# Packing: 괄호 없어도 콤마(,)만 있으면 튜플임
numbers = 1, 2, 3 

# Unpacking: 튜플을 풀어서 변수에 쏙쏙
a, b, c = numbers
print(a) # 1
print(b) # 2
```

#### [주의] 개수가 안 맞으면 에러! (ValueError)

변수 개수와 데이터 개수가 **정확히 일치**해야 합니다.

```python
nums = (1, 2, 3)

# (X) 변수가 부족해! (ValueError: too many values to unpack)
# a, b = nums 

# (X) 변수가 너무 많아! (ValueError: not enough values to unpack)
# a, b, c, d = nums
```

**팁: 남는 거 다 담기 (`*`)** 변수 앞에 별(`*`)을 붙이면 "나머지 전부 다 여기로 와!"가 됩니다

```python
a, *b = (1, 2, 3, 4)
# a는 1, b는 [2, 3, 4] (리스트로 담김)
```


---
## 리스트 정렬하기 (Sorting) 

순서를 섞거나 정리할 때 씁니다. (자세한 건 [[Python_Sorting_Logic#① `sort()` vs `sorted()` (면접 단골)|sort vs sorted]] 참고)

### ① `.sort()` : 원본 파괴 (주의!)

```python
nums = [3, 1, 2]
nums.sort()  
print(nums) # [1, 2, 3] (원본이 바뀜)
```

### ② `sorted()` : 원본 유지 (안전!)

```python
nums = [3, 1, 2]
new_nums = sorted(nums)
print(nums)     # [3, 1, 2] (원본 그대로)
print(new_nums) # [1, 2, 3] (정렬된 새 리스트)
```

---
## Deep Dive: Slicing (슬라이싱) 

리스트의 **일부분을 잘라내거나 복사**할 때 쓰는 핵심 기술입니다. 
Airflow에서 파일 경로 다룰 때 필수입니다.

- **문법:** `{python}리스트[시작 : 끝 : 간격]` (끝 번호 앞까지만 잘림!)

```python
data = [0, 1, 2, 3, 4, 5]

# 1. 자르기 (Start ~ End-1)
print(data[0:3]) # [0, 1, 2] (3은 포함 안 됨!)

# 2. 앞/뒤 생략 (처음부터/끝까지)
print(data[:3])  # [0, 1, 2] (처음부터 3 전까지)
print(data[3:])  # [3, 4, 5] (3부터 끝까지)

# 3. 전체 복사 (Copy) ✨ [중요]
# 그냥 `new = org` 하면 '참조(바로가기)'만 돼서 원본이 같이 바뀜.
# `new = org[:]` 해야 완전히 새로운 쌍둥이를 만듦.
copy_list = data[:]
```

---
## List Comprehension (리스트 컴프리헨션) 

 `for` 문과 `append`를 **한 줄**로 줄여쓰는 파이썬만의 강력한 문법입니다.

> 리스트에 담긴 데이터를 한 줄로 필터링하거나 변환하고 싶다면? 다중 필터링을 하고 싶다면?
>  👉 [[Python_List_Comprehension]] (리스트 컴프리헨션 심화) 참고해서 더 심화 보기 

- 문법 : `{python}[ 변환식 for 변수 in 리스트 ]`

### ① 기본 문법

- **일반 방식 (3줄)**

```python
nums = [1, 2, 3]
new_nums = []
for n in nums:
    new_nums.append(n * 10)
```

- **컴프리헨션 (1줄)**

```python
# [ 변환식 for 변수 in 리스트 ]
new_nums = [n * 10 for n in nums] 
# 결과: [10, 20, 30]
```

### ② 조건 걸기 (Filter)

- **상황:** 짝수만 뽑아서 리스트로 만들고 싶다.
- 문법 : `{python}[ 변환식 for 변수 in 리스트 if 조건 ]`

```python
# [ 변환식 for 변수 in 리스트 if 조건 ]
evens = [n for n in range(10) if n % 2 == 0]
# 결과: [0, 2, 4, 6, 8]
```

### ③ 조건에 따라 값 바꾸기 (If - Else) 

**"이거 아니면 저거를 넣겠다"** -> `if-else`가 **맨 앞**에 옵니다.
- 문법 :`{python}[ (참일때 값) if 조건 else (거짓일때 값) for 변수 in 리스트 ]`

```python
# [ (참일때 값) if 조건 else (거짓일때 값) for 변수 in 리스트 ]

# 짝수면 "even", 홀수면 "odd"를 채워라
results = ["even" if k % 2 == 0 else "odd" for k in range(5)]
# 결과: ['even', 'odd', 'even', 'odd', 'even']
```

#### [주의] `if` 위치가 헷갈려요!

- **뽑을까 말까 고민할 때 (Filter):** `if`는 **뒤**에! (`else` 못 씀)
	- `[x for x in data if x > 0]` (O)
	
- **A로 할까 B로 할까 고민할 때 (Choose):** `if-else`는 **앞**에!
	- `["양수" if x > 0 else "음수" for x in data]` (O)




---

## 초보자가 자주 착각하는 포인트 🚨

### ① "리스트 변수 그냥 복사하면 안 돼요?" (`{text}=`)

```python
list_a = [1, 2]
list_b = list_a  # 이건 복사가 아니라 '별명 붙이기'야!

list_b.append(3)
print(list_a) # [1, 2, 3] -> 헉, 원본도 바뀜! 
```

- **해결:** 반드시 `list_a[:]` 처럼 슬라이싱을 쓰거나 `.copy()`를 써야 독립적인 리스트가 됩니다.

### ② "항목이 하나일 때 튜플이 안 돼요!"

- 요소가 딱 하나만 있을 때는 **반드시 뒤에 콤마(,)** 를 찍어야 합니다.

```python
a = ("hello")  # 그냥 문자열(String) (수학 괄호 취급)
b = ("hello",) # 비로소 튜플(Tuple)

# "괄호가 본체가 아니라, 콤마(,)가 본체다!"
```



