---
aliases:
  - Set
  - 셋
  - 집합
  - 중복제거
  - len(set)
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
---
##  개념: 중복을 허용하지 않는 주머니 

`set`(집합)은 두 가지 큰 특징이 있습니다.

1.  **중복이 없다:** 같은 값을 백 번 넣어도 딱 하나만 남습니다.
2.  **순서가 없다:** 인덱싱(`[0]`)이 불가능합니다.

---
## 핵심 활용: 로직 단순화 (Logic Simplification) 

사용자님의 사례처럼 복잡한 비교 연산자(`!=`, `{text}==`) 범벅을 **집합의 크기(`len`)** 하나로 깔끔하게 해결할 수 있습니다.

### ❌ Before: 하드 코딩 (The Hard Way)

변수끼리 일일이 비교하느라 조건문이 길어지고, 오타가 나기 쉽습니다.

```python
def solution(a, b, c):
    answer = 0
    # 3개가 모두 다를 때
    if a != b and a != c and b != c:
        answer = a + b + c
    # 3개가 모두 같을 때
    elif a == b and b == c and a == c:
        answer = (a + b + c) * (a**2 + b**2 + c**2) * (a**3 + b**3 + c**3)
    # 2개만 같을 때 (조건이 가장 복잡함!)
    elif a != b == c or a == b != c or a == c != b:
        answer = (a + b + c) * (a**2 + b**2 + c**2)
    return answer
```


### After: Set 활용 (The Smart Way)

리스트를 집합으로 바꾸면 **중복이 사라진다**는 점을 이용합니다.

- **3개** 남음 (`{1, 2, 3}`) → 모두 다름
- **2개** 남음 (`{1, 2}`) → 두 개는 같음
- **1개** 남음 (`{1}`) → 모두 같음

```python
def solution(a, b, c):
    # 1. 리스트로 묶고 계산 준비
    nums = [a, b, c]
    s = sum(nums)
    
    # 2. 집합의 크기(len)로 상황 판단
    unique_count = len(set(nums))
    
    if unique_count == 3:      # 모두 다름
        return s
    elif unique_count == 2:    # 두 개만 같음
        return s * sum(x**2 for x in nums)
    else:                      # 모두 같음 (unique_count == 1)
        return s * sum(x**2 for x in nums) * sum(x**3 for x in nums)
```

**통찰:** 변수가 `a, b, c, d` 4개로 늘어난다면?

- **Before:** `if`문이 수십 줄로 늘어납니다.
- **After:** `len(set(nums))` 로직은 그대로 유지됩니다. (확장성 최고!)

---
## 데이터 엔지니어링 필수 연산 (집합 연산) 

데이터 비교 분석(Diff)을 할 때 필수적입니다.

```python
A = {1, 2, 3}
B = {2, 3, 4}

# 교집합 (공통된 데이터 찾기)
print(A & B)  # {2, 3}

# 차집합 (A에는 있는데 B에는 없는 것 = 누락 데이터 찾기)
print(A - B)  # {1}

# 합집합 (모두 합치기, 중복 제거됨)
print(A | B)  # {1, 2, 3, 4}
```

---
## 요약 (Cheat Sheet)

|**상황**|**코드**|**설명**|
|---|---|---|
|**중복 제거**|`list(set(data))`|리스트를 깨끗하게 청소|
|**다름/같음 판단**|`len(set(data))`|조건문 지옥 탈출 열쇠|
|**공통 값 찾기**|`set_a & set_b`|교집합 연산|


