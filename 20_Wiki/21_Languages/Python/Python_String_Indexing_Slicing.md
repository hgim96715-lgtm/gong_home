---
aliases:
  - String Indexing
  - Slicing
  - 문자열 슬라이싱
  - 문자열 인덱싱
  - 불변객체
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_String_Methods]]"
---
## 개념 한 줄 요약 ️

**"문자열의 특정 위치를 콕 집어내거나(`Indexing`), 범위를 지정해서 잘라내는(`Slicing`) 기술."**

* **핵심:** 파이썬의 문자열은 **변경 불가능(Immutable)** 합니다.
* **따라서:** `str[0] = 'a'` 처럼 직접 수정은 불가능하며, **슬라이싱으로 잘라서 새 문자열을 조립**해야 합니다.

---
## 인덱싱 (Indexing): 한 글자만 콕! 

파이썬은 **0부터 셉니다.** 그리고 **뒤에서부터 셀 때는 -1**을 씁니다.

```python
text = "PYTHON"
#  P  Y  T  H  O  N
#  0  1  2  3  4  5  (양수 인덱스)
# -6 -5 -4 -3 -2 -1  (음수 인덱스)

print(text[0])   # 'P' (맨 앞)
print(text[-1])  # 'N' (맨 뒤)
```

---
## 슬라이싱 (Slicing): 범위로 자르기 

**문법:** `{python}[시작 : 끝 : 간격]`

- **주의:** **"끝 인덱스는 포함하지 않습니다."** (End is Exclusive)
- `[start : end]` -> `start` 이상 `end` **미만**

### ① 기본 자르기 

```python
text = "Hello World"

# 인덱스 0부터 5 '전'까지 (= 0~4)
print(text[0:5])   # "Hello"

# 앞부분 생략 (= 처음부터)
print(text[:5])    # "Hello"

# 뒷부분 생략 (= 끝까지)
print(text[6:])    # "World"
```

### ② 간격(Step) 활용하기 ( \:\: )

```python
numbers = "123456789"

# 2칸씩 띄어서 가져오기
print(numbers[::2])  # "13579"

# [꿀팁] 문자열 뒤집기 (Reverse) ⭐️
print(numbers[::-1]) # "987654321"
```

### ③ 슬라이싱 연계 (Chaining) ️

슬라이싱을 두 번 연속으로 써서 **"자르고 + 뒤집기"** 를 한 방에 처리하는 기술입니다.
코딩 테스트에서 문자열 뒤집기 문제 나올 때 필살기처럼 쓰입니다.

> **💡 원리:** `text[start:end]`의 결과물도 **"문자열(String)"** 입니다. 그러니 그 뒤에 바로 `[:​:-1]`을 붙여도 파이썬은 찰떡같이 알아듣습니다.

```python
text = "Pro Python"

# 1단계: "Pro P"까지만 자르고 (text[:5])
# 2단계: 그걸 바로 뒤집어라 ([::-1])
# 결과: "P orP"
print(text[:5][::-1]) 

# [실전 응용]
# s 문자열의 a번째부터 b번째까지 자른 뒤 뒤집어라!
# (b+1은 인덱스가 0부터 시작하기 때문에, b번째 글자까지 포함하려고 더한 것)
answer = my_string[a : b+1][::-1]
```

---
## [중요] 문자열 수정하기 (Immutability) 

코딩 테스트에서 가장 많이 하는 실수입니다

### ❌ 틀린 방법 (직접 대입)

```python
s = "Teat"
# 2번째 글자 'a'를 's'로 바꾸고 싶다.
s[2] = 's' 
# 👉 에러 발생! (TypeError: 'str' object does not support item assignment)
```

### ⭕️ 맞는 방법 (오려 붙이기)

원본을 건드릴 수 없으니, `{python}New = Old[:idx] + "New_Char" + Old[idx+1:]`을 합쳐서 **새로운 문자열**을 만듭니다.

```python
s = "Teat"
target_index = 2  # 바꾸고 싶은 위치 ('a')

# 1. 앞부분 (Head): 처음부터 ~ 타겟 직전까지
front = s[:target_index]   # "Te" (0, 1번 인덱스)

# 2. 바꿀 부분 (New Body): 새로운 글자
middle = "s"               # "s"

# 3. 뒷부분 (Tail): 타겟 다음부터(~ +1) ~ 끝까지 ⭐️중요!
# (idx+1을 해야 원래 있던 'a'를 건너뛰고 't'부터 가져옵니다.)
back = s[target_index + 1:] # "t" (3번 인덱스부터 끝까지)

# 4. 봉합 (Concatenation)
result = front + middle + back
print(result) # "Test"
```

#### 주의할 점 (The Trap)

뒷부분을 자를 때 `s[target_index:]`라고 쓰면 **원래 글자가 또 들어갑니다!**
반드시 **`+1`** 을 해서 기존 글자를 **건너뛰어야(Skip)** 합니다.

- ❌ `s[2:]` → `"at"` ('a'가 살아있음)
- ⭕️ `s[2+1:]` → `"t"` ('a'가 제거됨)




>"리스트(`list`)와 문자열(`str`)은 둘 다 인덱싱/슬라이싱이 돼서 비슷해 보이지? 
>하지만 결정적인 차이가 바로 **'수정 가능 여부'** 야. 
>리스트는 `a[0] = 1`이 되지만, 문자열은 절대 안 돼. 
>이걸 **'불변 객체(Immutable Object)'** 라고 부르는데, 파이썬의 가장 중요한 특징 중 하나니까 꼭 기억해둬!"




