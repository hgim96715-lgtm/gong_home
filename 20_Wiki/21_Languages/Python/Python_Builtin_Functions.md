---
aliases:
  - Built-in Functions
  - max
  - min
  - sum
  - len
  - abs
  - 내장함수
  - ord
  - chr
  - pow
  - bit_length()
tags:
  - Python
related:
  - "[[Python_Sorting_Logic]]"
  - "[[Python_Variables_Types]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Math_Module]]"
---

## 개념 한 줄 요약 

**"파이썬이 이미 만들어 둔 도구들. `import` 없이 바로 쓸 수 있다!"**


- **집계(Aggregation):** 반복문(`for`) 없이 리스트의 합계, 최대/최소, 길이를 한 방에 구합니다. (`max`, `min`, `sum`, `len`)
- **변환(Conversion):** 컴퓨터가 문자를 숫자로 기억하는 방식(ASCII)을 이용해 변환합니다. (`ord`, `chr`)
- **핵심:** 굳이 `for`문이나 `if`문을 길게 쓰지 말고, 이 함수들을 먼저 떠올리는 것이 **Pythonic**한 코딩의 첫걸음입니다.

---
## Why is it needed (왜 필요한가?)

**코드 압축 (Efficiency):** `max()`를 쓰면 5~6줄짜리 `if a > b` 로직을 단 1줄로 줄일 수 있습니다. 가독성과 속도가 동시에 좋아집니다.
**컴퓨터와의 대화 (Low-level):** 컴퓨터는 'A'를 그림이 아니라 숫자(65)로 기억합니다. 암호화 알고리즘이나 엑셀 칼럼 계산 등에서 문자를 숫자로 다뤄야 할 때 `ord/chr`가 필수입니다.

---
## Practical Context (실전 활용)

- **데이터 파이프라인 (Batch Processing):** 대용량 데이터를 특정 사이즈로 나누어 처리할 때, 작업 횟수와 남는 데이터를 `divmod`로 한 번에 계산한다.
- **알고리즘 문제 (Greedy):** "가장 큰 수 만들기", "가장 작은 비용 찾기" 등 최적의 해를 찾을 때 `max`, `min`이 항상 쓰인다.
- **데이터 검증:** 입력받은 리스트가 비어있는지 확인할 때 `len(lst) > 0`을 사용합니다.
- **보안/암호화:** 'a'를 'b'로 바꾸는 시저 암호(Caesar Cipher)를 만들 때 `ord`로 숫자로 바꾼 뒤 +1을 하고 다시 `chr`로 문자로 바꾼다.

---
## Code Core Points

## 집계와 수학 (`max`, `min`, `sum`, `len`, `abs`)

```python
# 1. 최댓값 / 최솟값 (max, min)
 numbers = [1, 5, 3, 9] 
 print(max(numbers)) # 9 (리스트 통째로 넣기) 
 print(min(10, 20)) # 10 (인자 여러 개 넣기) 
 
 # 2. 합계 (sum) - for문 돌리지 마세요! 
 print(sum(numbers)) # 18 
 
 # 3. 길이 (len) 
 text = "Hello" 
 print(len(text)) # 5 
 
 # 4. 절댓값 (abs) 
 print(abs(-100)) # 100
```

> **작동 원리:** 내부적으로 C 언어로 구현된 최적화된 로직을 타기 때문에 직접 반복문을 작성하는 것보다 훨씬 빠르다.

---
## Quotient and Remainder (몫과 나머지: divmod)

나눗셈의 몫과 나머지를 튜플(Tuple) 형태로 한 번에 반환한다.

- **문법:** `divmod(a, b)` (a: 나누어지는 수, b: 나누는 수)

```python
total_records = 10050
batch_size = 1000

batches, remainder = divmod(total_records, batch_size)

print(f"전체 배치 수: {batches}, 남은 데이터: {remainder}") # 전체 배치 수: 10, 남은 데이터: 50
```

>**작동 원리:** `//` 연산과 `%` 연산을 따로 두 번 하는 것보다, `divmod`를 호출해 한 번에 처리하는 것이 성능상 유리하다.

---
## 거듭제곱 (`pow`)

`**` 연산자와 비슷하지만, **3번째 인자(mod)** 를 넣으면 훨씬 강력해집니다.

```python
# 1. 기본 사용법 (** 연산자와 동일)
print(pow(2, 10))    # 1024 (2의 10승)
print(2 ** 10)       # 1024 (같은 결과)

# 2. 3번째 인자: 모듈러 연산 (나머지)
# pow(base, exp, mod) = (base ** exp) % mod
# 단순히 ** 후 % 하는 것보다 훨씬 빠름!
print(pow(2, 10, 1000))  # 24  (1024 % 1000)

# [활용] 큰 수의 나머지 계산 (알고리즘 단골 패턴)
# 10억^10억 같은 엄청난 수도 한 방에 계산 가능
print(pow(10**9, 10**9, 10**9 + 7))  # 빠르고 정확하게 나머지 반환
```

>💡 왜 `pow(a, b, mod)`가 `(a**b) % mod` 보다 좋은가?
>`a**b`를 먼저 계산하면 중간에 천문학적으로 큰 수가 메모리를 잡아먹습니다.
>`pow(a, b, mod)`는 내부적으로 **빠른 거듭제곱(Fast Exponentiation)** 알고리즘을 사용해 중간 결과를 계속 mod로 줄이며 계산합니다.

---
## 비트 길이 (`int.bit_length()`)

`bit_length()`는 정수를 **2진수(이진법)** 로 표현할 때 필요한 **비트(자릿수)의 개수**를 반환합니다.

```python
# 10진수 → 2진수로 변환해서 생각하기
print(bin(1))   # '0b1'     → 비트 1개
print(bin(4))   # '0b100'   → 비트 3개
print(bin(7))   # '0b111'   → 비트 3개
print(bin(8))   # '0b1000'  → 비트 4개

# bit_length()는 앞의 '0b'를 빼고 유효 비트 수만 셉니다
print((1).bit_length())   # 1
print((4).bit_length())   # 3
print((7).bit_length())   # 3
print((8).bit_length())   # 4

# 규칙: n이 0이 아닐 때, 2^(n.bit_length()-1) <= n < 2^(n.bit_length())
```


### `bit_length()`로 "다음 2의 거듭제곱" 구하기

아래 코드는 배열 길이를 **2의 거듭제곱 크기**로 올림(패딩)하는 전형적인 패턴입니다.

```python
def solution(arr):
    length = len(arr)
    target = 1 << (length - 1).bit_length()
    return arr + [0] * (target - length)

# 예시로 분해해보기
arr = [1, 2, 3, 4, 5]  # length = 5

# Step 1: length - 1 = 4
# Step 2: (4).bit_length() = 3   (4는 이진수로 '100', 3비트)
# Step 3: 1 << 3 = 8             (1을 왼쪽으로 3칸 shift = 2^3 = 8)
# Step 4: arr + [0] * (8 - 5) = [1, 2, 3, 4, 5, 0, 0, 0]

print(solution([1, 2, 3, 4, 5]))   # [1, 2, 3, 4, 5, 0, 0, 0]
print(solution([1, 2, 3, 4]))      # [1, 2, 3, 4]  ← 이미 2의 거듭제곱이면 그대로!
```

**`<<` 연산자 (Left Shift) 란?**

|표현식|의미|결과|
|---|---|---|
|`1 << 0`|1 × 2⁰|1|
|`1 << 1`|1 × 2¹|2|
|`1 << 2`|1 × 2²|4|
|`1 << 3`|1 × 2³|8|
>`1 << n`은 **2ⁿ을 구하는 가장 빠른 방법**입니다.  
>`pow(2, n)` 또는 `2**n` 과 결과는 같지만, 비트 연산이라 속도가 더 빠릅니다.

---
## 문자 ↔ 숫자 변환 (`ord`, `chr`)

컴퓨터는 문자를 **아스키(ASCII) 코드**라는 숫자로 기억합니다.
문자와 아스키(ASCII) / 유니코드(Unicode) 숫자 간의 변환을 수행한다.

```python
# 1. 문자 → 숫자 (Ordinal Number)
# "문자 'a'의 번호표를 줘"
print(ord('a'))  # 97
print(ord('A'))  # 65 (대문자가 숫자가 더 작음!)

# 2. 숫자 → 문자 (Character)
# "번호표 97번인 문자 나와"
print(chr(97))   # 'a'

# [활용] 알파벳 A부터 C까지 출력하기
# range()는 숫자만 받으므로 변환이 필요함
for i in range(ord('A'), ord('D')): 
    print(chr(i), end=" ")
# 결과: A B C
```

>대문자('A'=65)가 소문자('a'=97)보다 숫자가 더 작다는 점을 기억해두면 정렬 로직을 짤 때 유용하다. 
>파이썬은 유니코드를 지원하므로 한글(`ord('가') = 44032`)도 정상적으로 변환된다.

---
## Detailed Analysis (심화 분석)

### 1. `max()`의 문자열 비교 로직 (사전순)

`max()` 안에 숫자가 아닌 **문자열**을 넣으면 **"사전 순서상 뒤에 나오는 것"** 을 크다고 판단합니다.

```python
# 1. 일반적인 사전 순서
print(max("apple", "banana")) # "banana" ('b'가 'a'보다 뒤니까)

# 2. 문자열 합치기 문제 (코딩테스트 꿀팁)
# 두 수를 합쳐서 더 큰 수를 만들고 싶다면?
a, b = 3, 30
str1 = f"{a}{b}" # "330"
str2 = f"{b}{a}" # "303"

# 문자열 상태에서 비교 (3이 0보다 크므로 330이 선택됨)
print(max(str1, str2)) # "330"
```

### 2. 숫자로 된 문자열 비교 주의 (함정 카드 ☠️)

문자열 비교는 **맨 앞글자**부터 따집니다. 숫자의 실제 크기(`value`)와 다릅니다!

```python
print(max(10, 9))     # 10 (숫자는 당연히 10이 큼)
print(max("10", "9")) # "9"  
```

- **이유:** 문자열 "9"의 첫 글자 `'9'`(아스키 57)가 "10"의 첫 글자 `'1'`(아스키 49)보다 크기 때문입니다.
- **해결:** 숫자 크기를 비교하려면 반드시 `int()`로 변환 후 비교하세요.

---
## Common Beginner Misconceptions (자주 하는 실수)

### ① `for`문 안에 `max/min` 넣지 마세요

`max` 함수 자체가 이미 데이터를 훑어보는 반복문입니다.
반복문 안에 넣으면 비효율적일 뿐만 아니라 에러가 날 수 있습니다.

```python
arr = [5, 3, 9, 1]

# [❌ 오답] 하나씩 꺼내서(c) max에 넣음 -> 에러 발생
# for c in arr:
#     print(max(c)) # TypeError: 'int' object is not iterable

# [✅ 정답] 리스트 통째로 던지세요
print(max(arr)) # 9
```

### ② `ord()`에는 딱 한 글자만!

`ord` 함수는 문자열 전체를 숫자로 바꿔주지 않습니다.

```python
# [❌ 오답]
# print(ord("ABC")) # TypeError 발생

# [✅ 정답]
print(ord("A")) # 65
```

### ③ "한글은 안 되나요?"

됩니다! 파이썬은 유니코드(Unicode)를 지원하므로 한글도 고유 번호가 있습니다.

```python
print(ord('가')) # 44032
```

### ④ `bit_length()`는 0일 때 0을 반환

0은 2진수로 표현할 유효 비트가 없기 때문에 0을 반환합니다. 
`length - 1` 패턴에서 `length = 1`일 때 `(0).bit_length() = 0`이 되어 `1 << 0 = 1`로 올바르게 처리됩니다.

```python
print((0).bit_length())  # 0
print((1).bit_length())  # 1
```

