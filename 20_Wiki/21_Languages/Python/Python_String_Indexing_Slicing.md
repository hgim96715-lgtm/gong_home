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

원본을 건드릴 수 없으니, **앞부분 + 바꿀 글자 + 뒷부분**을 합쳐서 **새로운 문자열**을 만듭니다.

```python
s = "Teat"

# 'Te' (앞부분) + 's' (교체) + 't' (뒷부분)
new_s = s[:2] + 's' + s[3:]

print(new_s) # "Test"
```

>"리스트(`list`)와 문자열(`str`)은 둘 다 인덱싱/슬라이싱이 돼서 비슷해 보이지? 
>하지만 결정적인 차이가 바로 **'수정 가능 여부'** 야. 
>리스트는 `a[0] = 1`이 되지만, 문자열은 절대 안 돼. 
>이걸 **'불변 객체(Immutable Object)'** 라고 부르는데, 파이썬의 가장 중요한 특징 중 하나니까 꼭 기억해둬!"




