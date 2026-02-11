---
aliases:
  - 반복자
  - Iterable
  - Iterator
  - 반복 가능한 객체
tags:
  - Python
related:
  - "[[Python_Generators_Yield]]"
  - "[[PyFlink_Evictor]]"
---
# Python Iterables & Iterators: 데이터 줄 세우기

## 한줄 요약

**Iterable**은 `for`문에 넣을 수 있는 "데이터 보따리"이고,
**Iterator**는 그 보따리에서 데이터를 "하나씩 꺼내주는 손가락(포인터)"입니다.


> **Iterable(이터러블)** 이란 `for` 문에 넣어서 내부 요소들을 하나씩 꺼내 쓸 수 있는 모든 객체를 말합니다(도시락 통)
> **Iterator:** "내용물"이 아니라 "꺼내는 동작"입니다!(젓가락)

---
## 왜 필요한가?

**메모리 폭발 방지 (Memory Efficiency):**
- 1억 개의 데이터가 담긴 리스트를 한 번에 메모리에 올리면 서버가 터집니다.
- 하지만 **Iterator**를 쓰면 "지금 당장 처리할 1개"만 메모리에 올려서 효율적으로 작업할 수 있습니다.

**게으른 계산 (Lazy Evaluation):**
- 미리 다 만들어두지 않고, 필요할 때(`next()`)만 계산해서 가져옵니다. 
- 끝을 알 수 없는 실시간 데이터 스트림(PyFlink) 처리에 필수적입니다.

**프로토콜 통일:**
- 파이썬은 `__iter__`와 `__next__`라는 약속(Protocol)을 통해 리스트, 튜플, 딕셔너리, 파일 등을 모두 동일한 방식으로 반복할 수 있게 만듭니다.

---
## 실무 핵심

PyFlink의 `ProcessWindowFunction`에서 `elements`를 받을 때 바로 이 개념이 쓰입니다.
Flink는 윈도우 안의 데이터를 리스트로 주지 않고 **Iterator** 형태로 줍니다.
그래야 수만 개의 데이터가 쌓여도 메모리를 아끼며 하나씩 계산할 수 있기 때문입니다.

---
## Code Core Points

- **`iter(obj)`**: `obj`가 가진 `__iter__` 메서드를 호출해 Iterator로 변환합니다.
- **`next(it)`**: Iterator의 `__next__` 메서드를 호출해 다음 데이터를 가져옵니다. 데이터가 없으면 `StopIteration` 예외를 던집니다.
-  **`list(it)`**: Iterator를 강제로 리스트로 변환합니다. 이때 Iterator 안에 남은 모든 데이터를 **메모리에 몽땅 적재**하므로 매우 주의해야 합니다.
- **일회용성**: Iterator는 한 번 끝까지 가면 다시 처음으로 돌아갈 수 없는 "일방통행"입니다.

- **종류:** 리스트(`list`), 튜플(`tuple`), 문자열(`str`), 딕셔너리(`dict`), `range()` 등이 모두 Iterable입니다.
- **판별법:** `iter()` 함수를 넣었을 때 에러가 안 나고 **Iterator(손가락)**를 뱉어내면 Iterable입니다.

---
## 코드분석

```python
# 1. 가장 흔한 Iterable: 리스트
# 보따리 안에 1, 2, 3이 줄 서서 기다리고 있습니다.
my_list = [1, 2, 3]

# 2. Iterable이기 때문에 for문에 바로 넣을 수 있습니다.
# 파이썬이 내부적으로 "보따리에서 하나씩 꺼내줘!"라고 명령합니다.
for item in my_list:
    print(item)

# 3. 문자열도 Iterable입니다.
# 한 글자씩 줄을 서 있는 상태죠.
for char in "Hello":
    print(char) # H, e, l, l, o가 순서대로 출력됨

# 1. 보따리(Iterable) 준비
lunch_box = ["Sandwich", "Apple", "Juice"]

# 2. 손가락(Iterator) 생성
# lunch_box 자체가 바뀌는 게 아니라, lunch_box를 가리키는 '손가락'이 생깁니다.
waiter = iter(lunch_box)

# 3. waiter는 "Sandwich" 그 자체가 아닙니다.
print(waiter) 
# 결과: <list_iterator object at 0x...> (점원 객체라고 나옵니다.)

# 4. 점원에게 "다음 거 줘!"라고 시킵니다.
print(next(waiter)) # Sandwich (이제야 내용물이 나옵니다!)
print(next(waiter)) # Apple (점원은 다음 위치를 기억하고 있습니다.)

```

- **`iter()`**: 데이터 보따리에서 전담 점원(Iterator)을 고용하는 명령입니다.
- `next()`: 점원에게 다음 물건을 꺼내오라고 시키는 명령입니다.

---
## 주의사항

- **"Iterator는 리스트랑 똑같은 거 아닌가요?"**
    - 아니요! 리스트는 모든 데이터를 **이미** 가지고 있는 사물함이고, Iterator는 요청할 때마다 데이터를 **꺼내주는 동작**에 집중합니다. 
    - 인덱스 접근(`it[0]`)이 안 된다는 점이 가장 큰 차이입니다.
        
- **"한 번 다 읽은 Iterator를 다시 `for`문에 돌려도 되나요?"**
    - 안 됩니다! Iterator는 한 번 소진되면 빈 껍데기가 됩니다. 
    - 다시 읽으려면 `iter()` 함수로 새 Iterator를 만들어야 합니다.

- **"왜 Flink는 번거롭게 리스트가 아니라 Iterator로 주나요?"**
    - 스트리밍 환경에서는 윈도우에 데이터가 얼마나 쌓일지 모릅니다. 
    - 수백만 건이 올 수도 있는데 리스트로 줬다가 서버가 즉사하는 걸 막으려는 **Flink의 배려(Safety)** 입니다.

- **"list(elements)를 하면 elements 안의 데이터는 어떻게 되나요?"**
	- `list()` 함수가 내부적으로 Iterator가 끝날 때까지 `next()`를 호출해버립니다. 
	- 따라서 `list(elements)`를 수행하고 난 뒤의 `elements`는 텅 비어버리게 됩니다.