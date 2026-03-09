---
aliases:
  - List
  - Tuple
  - 리스트
  - 튜플
  - Slicing
  - 슬라이싱
  - 배열
  - 사전식비교
  - insert
  - pop
  - append
tags:
  - Python
related:
  - "[[Python_Control_Flow]]"
  - "[[]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Variables_Types]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_String_Indexing_Slicing]]"
  - "[[Python_Unpacking]]"
  - "[[Python_Membership_In]]"
  - "[[Python_Looping_Helpers]]"
---
# Python_Lists_Tuples

## 개념 한 줄 요약

> **"데이터를 순서대로 담는 기차 같은 자료구조."**

|구분|List `[]`|Tuple `()`|
|---|---|---|
|**비유**|화물 기차 (짐을 뺐다 꼈다 가능)|밀봉된 기차 (출발하면 수정 불가)|
|**특징**|Mutable (수정 가능)|Immutable (수정 불가)|
|**용도**|데이터 수집, 가공, 변환|설정값, 좌표, 함수 리턴값|

---

---

# ① 생성과 조회

```python
tasks = ["extract", "transform", "load"]

tasks[0]    # "extract"   <- 앞에서 첫 번째 (0부터 시작!)
tasks[1]    # "transform" <- 앞에서 두 번째
tasks[-1]   # "load"      <- 뒤에서 첫 번째
tasks[-2]   # "transform" <- 뒤에서 두 번째
len(tasks)  # 3
```

```
인덱스 구조:
  ["extract",  "transform",  "load"]
       0            1           2      <- 앞에서
      -3           -2          -1      <- 뒤에서
```

---

---

# ② 추가 — append / insert / extend / +

```python
a = [1, 2, 3]

# append: 맨 뒤에 추가
a.append(4)
print(a)  # [1, 2, 3, 4]

# append: 리스트를 넣으면 통째로 들어감
a.append([5, 6])
print(a)  # [1, 2, 3, 4, [5, 6]]  <- 리스트 안에 리스트!

# insert: 원하는 위치에 삽입
a = [1, 2, 3]
a.insert(0, 99)   # 0번 자리에 99 삽입 (맨 앞에 추가)
print(a)  # [99, 1, 2, 3]

a.insert(2, 88)   # 2번 자리에 88 삽입
print(a)  # [99, 1, 88, 2, 3]

# extend: 리스트의 알맹이만 풀어서 합치기
a = [1, 2]
a.extend([3, 4])
print(a)  # [1, 2, 3, 4]

# +: 원본 유지, 새 리스트 반환
a = [1, 2]
c = a + [3, 4]
print(a)  # [1, 2]       <- 원본 그대로
print(c)  # [1, 2, 3, 4] <- 새 리스트
```

|방식|위치|동작|원본 변경|반환값|
|---|---|---|---|---|
|`append(x)`|맨 뒤|x 를 통째로 추가|O|None|
|`insert(i, x)`|i 번 자리|x 를 i 위치에 삽입|O|None|
|`extend(b)`|맨 뒤|b 의 알맹이만 풀어서|O|None|
|`a + b`|—|알맹이 합쳐서 새것 생성|X|새 리스트|

> `mix = a.extend(b)` 하면 `mix` 는 `None` extend 는 원본을 고치는 것이지 반환하지 않는다.

---

---

# ③ 삭제 — pop / del / remove / [:-n]

| 구분      | `pop()` | `pop(0)` | `del`  | `remove()` | `clear()` | `[:-n]`   |
| ------- | ------- | -------- | ------ | ---------- | --------- | --------- |
| **기능**  | 맨 뒤 꺼내기 | 맨 앞 꺼내기  | 위치로 삭제 | 값으로 삭제     | 전체 비우기    | 뒤에서 n개 제거 |
| **반환값** | 꺼낸 값    | 꺼낸 값     | 없음     | 없음         | 없음        | 새 리스트     |

```python
data = ["A", "B", "C", "D"]

# pop(): 맨 뒤 꺼내기 (Stack 패턴)
last = data.pop()
print(last)  # "D"
print(data)  # ["A", "B", "C"]

# pop(0): 맨 앞 꺼내기 (Queue 패턴)
first = data.pop(0)
print(first)  # "A"
print(data)   # ["B", "C"]
# 뒤 데이터가 전부 한 칸씩 당겨지므로 데이터 많을 때 느림 -> deque 권장

# pop(i): 특정 위치 꺼내기
data = ["A", "B", "C", "D"]
mid = data.pop(1)
print(mid)   # "B"
print(data)  # ["A", "C", "D"]

# del: 위치로 그냥 삭제 (반환값 없음)
data = ["A", "B", "C"]
del data[0]
print(data)  # ["B", "C"]

del data[0:2]  # 슬라이싱으로 여러 개 한 번에 삭제
print(data)    # []

# remove: 값으로 찾아서 삭제
data = ["Red", "Green", "Blue"]
data.remove("Green")
print(data)  # ["Red", "Blue"]
# 없는 값 remove 하면 ValueError 발생! -> if 로 먼저 확인

# clear: 전체 비우기 (빈 리스트로 만들기)
data = [1, 2, 3]
data.clear()
print(data)  # []
# del data[:] 와 동일하지만 clear() 가 더 명확

# [:-n] 슬라이싱: 뒤에서 n개 한 번에 제거
answer = [1, 2, 3, 4, 5]
v = 2
if v > 0:
    answer = answer[:-v]
print(answer)  # [1, 2, 3]
# v = 0 이면 answer[:0] = [] 가 되므로 조건 분기 필요
```

## pop 패턴 정리

```text
data.pop()   <- 맨 뒤 꺼내기 = Stack (후입선출 LIFO)
data.pop(0)  <- 맨 앞 꺼내기 = Queue (선입선출 FIFO)
data.pop(i)  <- i 번째 꺼내기

Stack 예: 실행 취소(Undo), 괄호 짝 검사
Queue 예: 프린터 대기열, 메시지 큐
```

---

---

# ④ 탐색 — count / in / index / enumerate

## count — 몇 개인지 세기

```python
votes = ["Apple", "Banana", "Apple", "Kiwi", "Apple"]

votes.count("Apple")   # 3
votes.count("Orange")  # 0  <- 없어도 에러 없이 0 반환
```

> 단순히 "있냐 없냐" 확인은 `in` 연산자가 더 빠름. `count` 는 끝까지 전부 훑기 때문.

```python
"Apple" in votes   # True  <- 있냐 없냐만 볼 땐 이게 빠름
```

---

## index — 위치 찾기 (자주 까먹는 것) ⭐️

```python
data = ["A", "B", "C", "A"]

data.index("A")  # 0  <- 맨 앞 하나만 반환
data.index("C")  # 2
# data.index("Z") -> ValueError!
# 반드시 in 으로 존재 확인 후 사용
```

### 핵심 패턴 — 정렬 후 index 로 순위 찾기

```
문제 상황:
  응급도를 내림차순으로 정렬하는 것까지는 생각했지만,
  정렬된 리스트에서 현재 값의 순위를 찾는 방법을 떠올리지 못했다.

해결:
  sorted() 로 정렬된 복사본을 만들고
  원본 값을 .index() 로 정렬본에서 찾으면 -> 그 위치가 순위
```

```python
scores = [40, 10, 30, 20]

# 1. sorted 로 내림차순 정렬 복사본 생성 (원본 유지)
sorted_scores = sorted(scores, reverse=True)
# -> [40, 30, 20, 10]

# 2. 원본을 순회하면서 index 로 순위 조회
for score in scores:
    rank = sorted_scores.index(score) + 1  # 0부터 시작 -> +1
    print(f"{score} -> {rank}등")

# 40 -> 1등
# 10 -> 4등
# 30 -> 2등
# 20 -> 3등
```

```
흐름 정리:
sorted_scores = [40, 30, 20, 10]
인덱스 번호        0   1   2   3

scores 의 40 -> sorted_scores.index(40) = 0 -> 0+1 = 1등
scores 의 10 -> sorted_scores.index(10) = 3 -> 3+1 = 4등
scores 의 30 -> sorted_scores.index(30) = 1 -> 1+1 = 2등
```

> sorted() 자세한 옵션은 [[Python_Sorting_Logic]] 참고

> 동점자 주의: 같은 값이 여러 개면 `index()` 는 가장 앞 위치만 반환. 동점자를 같은 순위로 처리할 때는 이 패턴으로 충분.
>  동점자를 각각 다른 순위로 처리해야 하면 `enumerate` 패턴 사용.

### 뒤에서부터 찾기

```python
data = ["A", "B", "C", "A"]

# 마지막 "A" 의 원래 인덱스 구하기
last_idx = len(data) - 1 - data[::-1].index("A")
# -> 3
```

---

## enumerate — 모든 위치 찾기

> `index()` 는 맨 처음 하나만 반환. 전부 찾으려면 `enumerate` + 리스트 컴프리헨션 조합.

```python
data = ["A", "B", "C", "A", "B", "A"]

locations = [i for i, val in enumerate(data) if val == "A"]
# -> [0, 3, 5]
```

> 리스트 컴프리헨션 자세한 내용 -> [[Python_List_Comprehension]] 참고

---

---

# ⑤ 정렬 — sort / sorted

```python
nums = [3, 1, 2]

# sort(): 원본 파괴
nums.sort()
print(nums)  # [1, 2, 3]

# sorted(): 원본 유지, 새 리스트 반환
nums = [3, 1, 2]
new_nums = sorted(nums)
print(nums)      # [3, 1, 2]  <- 원본 그대로
print(new_nums)  # [1, 2, 3]
```

> 자세한 정렬 옵션 -> [[Python_Sorting_Logic]] 참고

---

---

# ⑥ 슬라이싱 — [시작:끝:간격]

> **끝 번호는 포함하지 않는다.** (End Index is Exclusive)

```python
data = [0, 1, 2, 3, 4, 5]

data[:3]    # [0, 1, 2]           <- 처음부터 3개
data[3:]    # [3, 4, 5]           <- 3번 인덱스부터 끝
data[-2:]   # [4, 5]              <- 뒤에서 2개
data[:]     # [0, 1, 2, 3, 4, 5] <- 전체 복사 (원본 독립)
data[::-1]  # [5, 4, 3, 2, 1, 0] <- 전체 뒤집기
```

## 길이로 자르기

```python
text = "20240210_Data_Log"
s, l = 0, 4
print(text[s : s + l])  # "2024"
```

## 뒤에서 n개 가져오기

```python
text = "abcde"
n = 3

# 흔한 실수
print(text[-1 : -n-1])
# -> "" (빈 문자열)
# 이유: 시작(-1=e) 에서 step 기본값 +1(오른쪽) 인데
#       도착점(-n-1=b) 은 왼쪽 -> 도달 불가

# 해결: 시작점을 앞으로 당기기 (추천)
print(text[-n:])  # "cde"
```

## 특정 구간만 뒤집기

```python
def solution(my_string, s, e):
    return my_string[:s] + my_string[s:e+1][::-1] + my_string[e+1:]
# 앞부분 그대로 + 구간만 뒤집기 + 뒷부분 그대로
```

---

---

# ⑦ Tuple (튜플)

```python
db_info = ("localhost", 5432)

db_info[0]    # "localhost"
# db_info[0] = "127.0.0.1"  -> TypeError 발생
```

## Unpacking

```python
numbers = 1, 2, 3        # 괄호 없어도 콤마면 튜플
a, b, c = numbers        # 풀어서 변수에 담기

a, *b = (1, 2, 3, 4)     # a=1, b=[2, 3, 4]  <- 나머지 전부 *
```

> 요소가 하나일 때는 반드시 콤마 필요 `("hello")` -> 문자열 / `("hello",)` -> 튜플

---

---

# ⑧ 사전식 비교 (Lexicographical Comparison)

> 앞에서부터 차례대로 비교, 처음으로 다른 값이 나오는 순간 결과 확정.

```python
date1 = [2024, 1, 31]
date2 = [2024, 2, 1]

# 2024 == 2024 -> 다음 / 1 < 2 -> True 확정
print(date1 < date2)  # True

def solution(date1, date2):
    return int(date1 < date2)  # True -> 1, False -> 0
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|`list_b = list_a` 후 원본이 바뀜|참조 복사 (별명 붙이기)|`list_a[:]` 또는 `.copy()`|
|`mix = a.extend(b)` 가 None|extend 는 None 반환|`a.extend(b)` 후 `a` 사용|
|`text[-1:-n-1]` 이 빈 문자열|step 기본값 +1 방향 불일치|`text[-n:]` 사용|
|`index()` 에서 ValueError|없는 값 조회|`if val in data:` 로 먼저 확인|
|정렬 후 순위 못 구함|index() 패턴을 모름|`sorted_list.index(값) + 1`|
|요소 1개 튜플이 문자열로 됨|콤마 누락|`("hello",)` 콤마 필수|