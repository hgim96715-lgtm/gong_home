---
aliases:
  - Built-in Functions
  - max
  - min
  - sum
  - len
  - abs
  - 내장함수
tags:
  - Python
related:
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Variables_Types]]"
---

## 개념 한 줄 요약 ⚡️

**"파이썬이 이미 만들어 둔 도구들. `import` 없이 바로 쓸 수 있다!"**

* 굳이 `for`문 돌리거나 `if`문 쓰지 말고, 내장 함수를 먼저 떠올리는 것이 **Pythonic**한 코딩의 첫걸음입니다.

---
##  최댓값/최솟값 (`max` / `min`)

사용자님이 발견한 것처럼 `if-else` 로직을 한 단어로 줄여줍니다.


### ① 숫자 비교 (기본)

```python
# Old Style (직접 비교)
a, b = 10, 20
if a > b:
    print(a)
else:
    print(b)

# Pythonic Style ✨
print(max(a, b))  # 20
```


### ② 문자열 비교 (사전순) 

`max()`안에 문자열을 넣으면 **"사전 순서상 뒤에 나오는 것"** 을 큽니다고 판단합니다. (ASCII 코드 기준)

```python
# 1. 길이가 같을 때: 앞글자부터 비교
print(max("330", "303")) 
# 결과: "330" (둘째 자리 '3'이 '0'보다 크니까)

# 2. 문자열 합치기 문제에서의 활용
a, b = 3, 30
str1 = f"{a}{b}" # "330"
str2 = f"{b}{a}" # "303"

# 문자열 상태에서 max를 뽑고 -> 나중에 int로 변환
print(int(max(str1, str2))) # 330
```

**주의할 점 (함정 카드)** 
문자열 비교는 **맨 앞글자부터** 따집니다. 숫자 크기와 다릅니다!

```python
print(max(10, 9))     # 10 (당연함)
print(max("10", "9")) # "9" (충격! 😱)
```

- 이유: 문자열 "9"의 첫 글자 '9'가 "10"의 첫 글자 '1'보다 크기 때문입니다.
- 해결: 숫자의 크기를 비교하려면 반드시 `int()`로 변환 후 비교하세요.

---
## 기타 필수 내장 함수

### ① 길이 구하기 (`len`)

```python
s = "hello"
lst = [1, 2, 3]
print(len(s))   # 5
print(len(lst)) # 3
```

### ② 합계 구하기 (`sum`)

`for`문으로 누적하지 마세요!

```python
numbers = [1, 2, 3, 4, 5]
print(sum(numbers)) # 15
```

### ③ 절댓값 구하기 (`abs`)

```python
print(abs(-10)) # 10
```

---
## 실전 활용 (Coding Test Tip) 

**문제:** 두 수 `a`, `b`를 붙였을 때 더 큰 수를 리턴하라.

```python
def solution(a, b):
    # 1. f-string으로 붙인다 (문자열이 됨)
    case1 = f"{a}{b}"
    case2 = f"{b}{a}"
    
    # 2. max()로 사전순으로 더 큰 문자열을 찾는다.
    # ("991" vs "919" -> "991" 선택됨)
    best_case = max(case1, case2)
    
    # 3. 숫자로 바꿔서 리턴
    return int(best_case)
    
    # [한 줄 요약]
    # return int(max(f"{a}{b}", f"{b}{a}"))
```

