---
aliases:
  - 파이썬 변수 교환
  - Swap
  - 튜플 언패킹
  - a,b=b,a
tags:
  - Python
  - Algorithm
  - Python_Test
related:
  - "[[Python_Lists_Tuples]]"
---
## 개념 한줄 요약 

**Swap(스왑)** 이란 두 변수의 값을 서로 맞바꾸는 것을 말합니다.
파이썬에서는 임시 변수(`temp`) 없이 **콤마(`,`)** 만으로 단 한 줄에 값을 교환할 수 있습니다. 이를 **Tuple Unpacking**이라고 부릅니다.

---
## Why: 왜 좋은가?

* **C/Java 스타일 (Bad in Python):**

```python
    temp = a
    a = b
    b = temp
    # 3줄이나 써야 하고, temp라는 쓸데없는 변수가 메모리를 차지함.
```

* **Python 스타일 (Good):**

```python
    a, b = b, a
    # 1줄로 끝. 직관적이고 빠름.
```

---
##  Code Core Points 

이 문법은 **배열(List)의 원소 위치를 바꿀 때** 가장 많이 쓰입니다.

```python
# 상황: i번째 요소와 j번째 요소를 서로 바꾸고 싶다.
arr = [10, 20, 30, 40]
i, j = 0, 3  # 0번(10)과 3번(40)을 바꾸자

# [핵심 문법] 우변(Right)이 먼저 평가되어 튜플이 되고, 좌변(Left)에 풀려서 들어감
arr[i], arr[j] = arr[j], arr[i]

print(arr)
# 결과: [40, 20, 30, 10]
```

---
## Detailed Analysis (동작 원리)

눈에는 안 보이지만 내부적으로는 **튜플(Tuple)** 이 생성되었다가 사라집니다.

1. `c_arr[j], c_arr[i]` (우변): 파이썬이 잠시 `(40, 10)` 이라는 **튜플**을 메모리에 만듭니다.
2. `c_arr[i], c_arr[j] = ...` (좌변): 튜플 `(40, 10)`을 다시 풀어서(Unpacking)
    - 첫 번째 값(40)은 `c_arr[i]`에,
    - 두 번째 값(10)은 `c_arr[j]`에 넣습니다.

---
## Practical Context (어디에 쓰나?)

- **정렬 알고리즘:** 버블 정렬, 선택 정렬, 퀵 정렬 구현 시 데이터 위치 변경.
- **피보나치 수열:** `a, b = b, a+b` 형태로 이전 값을 갱신할 때.
- **코딩 테스트:** "문자열 뒤집기", "배열 섞기" 등의 문제.