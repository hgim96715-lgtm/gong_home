---
aliases:
  - range
  - enumerate
  - zip
  - 반복문도구
  - 파이썬반복
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Control_Flow]]"
  - "[[PySpark_Session_Context]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약

**"반복문(`for`)을 더 똑똑하고 편하게 만들어주는 3가지 필수 아이템."**

1.  **`range()`**: 숫자 범위를 만들어줌. (0, 1, 2...)
2.  **`enumerate()`**: 번호표(Index)를 같이 붙여줌.
3.  **`zip()`**: 두 개의 리스트를 지퍼처럼 잠가서 합쳐줌.

---
## 숫자 제조기: `range()` 🔢

데이터 엔지니어링에서 **"테스트 데이터 100만 개 만들어봐"** 할 때 가장 먼저 쓰는 함수입니다.

### ① 기본 문법

`{python}range(시작, 끝, 보폭), 끝 숫자는 포함 안함`

```python
# range(시작, 끝, 보폭)
# * 끝 숫자는 포함하지 않음! (Stop - 1)

# 0부터 4까지 (총 5개)
r1 = range(5) 

# 2부터 10까지, 2칸씩 점프 (2, 4, 6, 8, 10)
r2 = range(2, 11, 2)
```

### ② 얘도 "게으른 녀석"이다? 

 `map`처럼, `range`도 숫자를 미리 다 만들어두지 않습니다. 
 **"달라고 할 때 하나씩 줍니다."** (메모리 절약)

```python
r = range(999999999999) 
# 이렇게 큰 숫자를 써도 컴퓨터가 안 멈춤! (메모리 안 먹음)

print(r)       # 출력: range(0, 999999999999) (안 보여줌)
print(list(r)) # 🚨 주의! 이러면 컴퓨터 멈춤 (메모리 폭발)
```

### 💡 실전 꿀팁: 조건문(if) 없애기

반복문 안에서 `if i % 2 == 0` 처럼 짝수/홀수를 거르는 로직은 `range`의 **step**을 쓰면 아예 없앨 수 있습니다.

**[Before] 초보자의 코드 (if문 남발)**

```python
# 1부터 n까지 홀수만 더하기
total = 0
for i in range(n + 1):
 if i % 2 == 1: # 굳이 하나하나 검사함 
	 total += i
```

[After] 파이써닉한 코드 (step 활용)

```python
# 시작점을 1로 잡고, 2칸씩 점프! (검사할 필요가 없음)
total = sum(range(1, n + 1, 2))
```

>**[[Python_List_Comprehension]]** : `[i*i for i in range(2, n+1, 2)]` 처럼 짝수 제곱을 한 줄로 만드는 법 참고.



---
## 번호표 붙이기: `enumerate()` 

로그 남길 때 필수입니다. **"몇 번째 데이터 처리 중인지"** 알아야 하니까요.

```python
names = ["Spark", "Airflow", "Docker"]

# (X) 촌스러운 방식
i = 0
for name in names:
    print(i, name)
    i += 1

# (O) 세련된 방식 (Index와 Value를 동시에!)
for i, name in enumerate(names):
    print(f"{i}번째 기술: {name}")
    
# 출력:
# 0번째 기술: Spark
# 1번째 기술: Airflow
# 2번째 기술: Docker
```

---
## 합체하기: `zip()` 

두 개 이상의 리스트를 **같은 인덱스끼리(병렬로)** 묶어줍니다. 마치 옷의 지퍼가 맞물리듯 동작합니다.

- **문법:** `{python}zip(iterable1, iterable2, ...)`
	- **인자:** 묶고 싶은 리스트(또는 튜플, 문자열 등)를 콤마(`,`)로 계속 넣을 수 있습니다.
	- **반환값:** 튜플들의 **'제조기(Iterator)'**를 반환합니다. (내용물이 바로 보이지 않음!)
- **특징:**
	- 같은 인덱스끼리 **튜플** `(a, b)`로 묶는다.
	- `dict()`로 감싸면 바로 **(Dictionary)** 이 된다.
	- 길이가 다르면 **짧은 쪽에 맞춰서 잘린다.**

### (형변환)확인 

`zip` 자체는 메모리만 잡고 있는 상태입니다. 
내용을 보려면 `list()`나 `dict()`로 포장을 뜯어야 합니다.

```python
names = ['A', 'B']
scores = [10, 20]

# 1. 그냥 출력하면? (실체 없음)
print(zip(names, scores)) 
# 결과: <zip object at 0x10...> (메모리 주소만 나옴)

# 2. 리스트로 변환 (튜플 쌍으로 보임)
print(list(zip(names, scores)))
# 결과: [('A', 10), ('B', 20)]

# 3. 딕셔너리로 변환 (키:값 매핑됨) 
print(dict(zip(names, scores)))
# 결과: {'A': 10, 'B': 20}
```

### ① 기초: 두 리스트 동시에 돌리기 (Parallel Iteration)

`range(len(list))` 같은 복잡한 인덱스 없이 깔끔하게 돕니다.

```python
names = ["Ironman", "Hulk"]
powers = [100, 999]

# Bad 👎 (인덱스 i를 직접 관리)
for i in range(len(names)):
    print(names[i], powers[i])

# Good 👍 (zip으로 묶어서 한 번에 언패킹)
for n, p in zip(names, powers):
    print(f"{n}의 파워는 {p}")
```

### ② 심화: 매핑 테이블 만들기 (Lookup Table) 🌟

입력값에 따라 더하거나 뺄 숫자가 정해져 있을 때, `if-elif`를 100줄 쓰는 대신 `dict(zip(...))` 한 줄로 끝냅니다.

**[상황]**
- 'w'는 +1, 's'는 -1, 'd'는 +10, 'a'는 -10을 해야 한다.

**[비효율적인 방식 (if문 지옥)]**

```python
# 하나하나 다 물어봐야 함 (느리고 김)
def solution_bad(n, control):
    for c in control:
        if c == 'w': n += 1
        elif c == 's': n -= 1
        elif c == 'd': n += 10
        elif c == 'a': n -= 10
    return n
```

[파이썬 고수의 방식 (zip + dict)]

```python
def solution_good(n, control):
    # 1. 키(Key)와 값(Value)을 따로 정의하고 zip으로 묶어버림 (매핑 테이블 생성)
    # 결과: {'w': 1, 's': -1, 'd': 10, 'a': -10}
    key = dict(zip(['w', 's', 'd', 'a'], [1, -1, 10, -10]))
    
    # 2. 제어 문자(c)를 키로 넣어서 바로 숫자를 꺼냄 (key[c])
    # sum()으로 변화량을 한 번에 더해버림
    return n + sum(key[c] for c in control)
```

>이렇게 하면 나중에 기능('e'키 추가)이 늘어나도 **리스트에 데이터만 추가**하면 됩니다. 
>코드를 뜯어고칠 필요가 없죠. (**Data-Driven**)
>`zip` + `dict`는  **조건문을 데이터로 치환**할 때 가장 강력한 패턴이다.
---
## Spark에서의 활용 (RDD 실습 복습)

```python
sc.parallelize( range(1000) )
```

- **`range(1000)`**: 0~999까지 숫자를 생성할 **준비**만 함. (메모리 거의 0)
- **`sc.parallelize(...)`**: 그 준비된 `range` 객체를 받아서, 여러 노드(컴퓨터)로 쫙 뿌리면서(Distributed) 실제 데이터를 만듦.



###  팁
> **`range`** 는 리스트가 아니라는 거! 
>`print(range(5))` 했을 때 `[0, 1, 2, 3, 4]`가 안 나온다고 당황하지 말고, **'아, 이 녀석도 게으른 녀석이구나, `list()`를 씌워야겠네'** 라고 생각하기

---
## 초기화 위치의 전략 (Scope: Outside vs Inside)

반복문을 돌 때 변수(빈 리스트 `[]`, 카운터 `0` 등)를 **어디서 초기화하느냐**에 따라 로직이 완전히 달라집니다. 
상황에 맞춰 골라 써야 합니다.

### ① 누적하기 (Accumulator): `Loop` 밖

>"전체 데이터를 훑어서 하나의 큰 결과를 만들 때"
- **목적:** 모든 바퀴의 기록을 **유지**해야 함. (합계, 전체 필터링)
- **위치:** `for`문 **시작 전(위)**.

```python
# [상황] 전체 숫자 중 3보다 큰 것만 다 모아줘!
c_arr = [1, 5, 10, 20]
res = []  # <--- [밖] 장바구니 하나로 끝까지 씀

for x in c_arr:
    if x > 3:
        res.append(x) # 계속 쌓임

print(res) 
# 결과: [5, 10, 20] (모두 누적됨)
```

### ② 리셋하기 (Resetter): `Loop` 안

> **"각각의 데이터마다 새로운 계산 결과가 필요할 때"**

- **목적:** 이번 바퀴의 계산이 끝나면, 다음 바퀴를 위해 **비워야(Reset)** 함.
- **위치:** `for`문 **내부(들어오자마자)**.

```python
# [상황] 각 '단어 별로' 철자를 쪼개줘! (단어마다 통이 새로 필요함)
words = ["Hi", "Bye"]

for w in words:
    char_list = []  # <--- [안] 단어 바뀔 때마다 새 식판 꺼냄!
    
    for c in w:
        char_list.append(c)
        
    print(f"단어: {w}, 분해: {char_list}")
    # 루프가 끝나면 char_list는 버려지고, 다음 단어 때 새로 만들어짐

# 결과:
# 단어: Hi, 분해: ['H', 'i']
# 단어: Bye, 분해: ['B', 'y', 'e'] (앞 단어랑 안 섞임!)
```

## 판단 기준 (Cheat Sheet)

| **질문**                           | **초기화 위치**      | **예시**                      |
| -------------------------------- | --------------- | --------------------------- |
| **"이전 바퀴의 결과가 다음 바퀴에 영향을 주는가?"** | **밖 (Outside)** | 총 합계(`total`), 전체 목록(`res`) |
| **"각 바퀴(행)마다 결과가 독립적인가?"**       | **안 (Inside)**  | 학생별 점수, 행별 합계, 단어별 길이       |