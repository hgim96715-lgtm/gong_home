---
aliases:
  - Python Generator
  - yield 키워드
  - 제너레이터
  - Lazy Evaluation
tags:
  - Python
related:
  - "[[Python_Functions]]"
  - "[[Python_Looping_Helpers]]"
  - "[[00_Python_HomePage]]"
---
#  메모리 효율의 왕: Generator & Yield

## 개념 한 줄 요약 (Concept Summary)

**"한 번에 하나씩만 생산해서 던져주는 공장."**
리스트처럼 데이터를 메모리에 몽땅 올려두는 게 아니라, **요청이 올 때마다 하나씩 만들어서(`yield`)** 건네주는 방식이다.

## 왜 필요한가 (Why)

* **메모리 절약:** 데이터가 100만 개일 때, 리스트(`return list`)는 100만 개를 메모리에 다 올려야 해서 컴퓨터가 뻗는다. 제너레이터(`yield`)는 딱 1개 분량의 메모리만 쓴다.
* **Flink/Streaming:** 데이터가 끝도 없이 들어오는 실시간 처리에서는 "끝"이 없으므로 리스트를 못 만든다. `yield`를 써야만 계속 흘려보낼 수 있다.

##  Practical Context (실무 활용)

* **Flink `flat_map`:** 문장을 단어로 쪼갤 때 `yield`를 써서 즉시 다음 단계로 넘긴다.
* **대용량 로그 파일 읽기:** 10GB짜리 로그 파일을 읽을 때 `readlines()`(리스트)를 쓰면 터진다. `yield`로 한 줄씩 읽어야 한다.

---
##  Code Core Points

### ① `return` vs `yield` 비교

```python
# [Bad] 리스트 방식 (메모리 많이 씀)
def make_list():
    result = []
    for i in range(5):
        result.append(i)
    return result  # [0, 1, 2, 3, 4]를 한 번에 반환

# [Good] 제너레이터 방식 (메모리 절약)
def make_generator():
    for i in range(5):
        yield i  # 0 던지고 대기 -> 1 던지고 대기 -> ...
```

### ② 사용법 (Iterate)

제너레이터는 `for` 문이나 `next()` 함수로 꺼내 쓸 수 있다.

```python
gen = make_generator()

print(next(gen)) # 0
print(next(gen)) # 1

# 보통은 이렇게 쓴다
for num in make_generator():
    print(num)
```

## Common Beginner Misconceptions (오해)

- **Q: 리스트랑 똑같이 생겼는데 뭐가 달라요?**
    - `type()`을 찍어보면 `list`가 아니라 `generator`라고 나온다.
    - **인덱싱 불가:** `gen[0]` 처럼 몇 번째 데이터를 바로 꺼낼 수 없다. (순서대로만 꺼내야 함)
    - **일회용:** 한 번 끝까지 다 읽으면("소진됨"), 다시 `for`문을 돌려도 아무것도 안 나온다. (다시 쓰려면 함수를 다시 호출해야 함)