---
aliases:
  - 대소문자변환
  - 문자열치환
  - replace
  - re.sub
  - swapcase
  - 정규식치환
  - upper
  - lower
  - group
tags:
  - python
related:
  - "[[Python_String_Methods]]"
  - "[[Python_Modules_Imports]]"
  - "[[Python_Regex]]"
  - "[[00_Python_HomePage]]"
---
##  개념 한 줄 요약 

**"단순한 글자 바꾸기는 `replace`를 쓰고, 복잡한 패턴(영어만, 숫자만 등)을 바꿀 땐 `re.sub`을 쓰는 것."**

 **대원칙:** 
* 파이썬의 `string` 객체는 **불변(Immutable)** 입니다. 
* 변환 메서드를 써도 원본은 바뀌지 않으니, 반드시 **변수에 재할당**해야 합니다.

---
## 왜 필요한가? (Why?) 

코딩 테스트나 데이터 전처리에서 가장 많이 하는 실수 중 하나가 **`replace()` 안에 정규식 패턴을 넣는 것**입니다.

* **흔한 실수:** `text.replace("[a-z]", "A")` 
    * 👉 이렇게 쓰면 파이썬은 진짜로 `[` `a` `-` `z` `]` 라는 글자 5개를 찾습니다.
* **해결책:** "패턴"을 인식시키려면 내장 모듈인 **`re` (Regular Expression)** 를 불러와야 합니다.
* **필요성:** 이 차이를 모르면 간단한 대소문자 변환 로직을 짤 때 `for`문과 `if`문을 남발하여 코드가 10줄 이상 길어집니다.

---
## Code Core Points (대소문자 변환 5대장 + 상태 확인)

`for`문을 돌며 `if char.isupper():` 처럼 하나하나 검사하지 않고, 통째로 바꾸는 것이 훨씬 빠르고 효율적이다.

### `.upper()`와 `.lower()` : 전체 일괄 변환

문자열 전체를 싹 다 대문자 혹은 소문자로 한 번에 바꾼다.

```python
s = "PyThon"

print(s.upper()) # 결과: "PYTHON" (싹 다 대문자로)
print(s.lower()) # 결과: "python" (싹 다 소문자로)
```

### `.swapcase()` : 대소문자 뒤집기

대문자는 소문자로, 소문자는 대문자로 반전시킨다.

```python
print(s.swapcase()) # 결과: "pYtHON"
```

### `.capitalize()`와 `.title()` : 앞글자만 대문자로

문장의 첫 글자만 대문자로 바꿀지, 아니면 띄어쓰기를 기준으로 각 단어의 첫 글자를 모두 대문자로 바꿀지 결정한다.

- `.capitalize()` : 문장 맨 앞 글자만 대문자로
- `.title()` : 각 단어마다 앞글자를 대문자로

```python
text = "hello python programming"

print(text.capitalize()) 
# 결과: "Hello python programming" (문장 맨 앞 글자만 대문자로)

print(text.title())      
# 결과: "Hello Python Programming" (각 단어마다 앞글자를 대문자로)
```


### `.isupper()`와 `.islower()` : 상태 확인용 (True/False)

해당 문자(또는 문자열 전체)가 대문자인지 소문자인지 판별하여 불리언(Boolean) 값을 반환

```python
print("A".isupper()) # 결과: True
print("a".islower()) # 결과: True
```


---
##  치환의 두 가지 세계 (`replace` vs `re.sub`) 

가장 중요한 부분입니다. 
**"무엇을"** 찾을 것인가에 따라 도구를 골라야 합니다.

### ① 단순 문자열 치환: `.replace()

* **언제 씀?** "사과"를 "배"로 바꾸고 싶을 때. (정확한 텍스트 매칭)
> `replace()`는 **"위치(Index)"가 아니라 "값(Value)"** 을 찾아서 바꾼다.!!!!!
* **특징:** 별도 `import` 필요 없음.

```python
text = "Hello World"

# "World"라는 글자를 찾아 "Python"으로 변경
new_text = text.replace("World", "Python") 
# 결과: "Hello Python"
```

#### `.replace()` 응용 : 반복문 없는 가로 확장 (문자열 곱셈)

문자열 곱셈(`*`)과 메서드 체이닝(Chaining)을 결합하면, `for`문을 쓰지 않고도 특정 문자를 원하는 배수만큼 가로로 확장시킬 수 있다.

```python
# '.'과 'x'를 각각 k배(여기서는 2배)로 늘리기
k = 2
row = ".x."

# '.'을 '..'으로, 'x'를 'xx'로 연달아 치환한다.
expanded_row = row.replace('.', '.' * k).replace('x', 'x' * k)

# 결과: "..xx.."
```


### ② 패턴(정규식) 치환: `re.sub()`

- **언제 씀?** "모든 소문자"를 대문자로, "모든 숫자"를 공백으로 바꾸고 싶을 때.
- **사용법:** `import re` 필수.
- **문법:** `{python}re.sub(패턴, 바꿀값, 원본텍스트)`

```python
import re

text = "abc123def"

# [패턴] 소문자([a-z])를 모두 찾아 "X"로 바꿔라
new_text = re.sub(r"[a-z]", "X", text)
# 결과: "XXX123XXX"

# [패턴] 숫자(\d)를 모두 없애라("")
clean_text = re.sub(r"\d", "", text)
# 결과: "abcdef"
```

> 정규식에 대한 자세한 내용은 [[Python_Regex]]참고 !

---
## 실전 꿀팁 (Coding Test Tip)

Q. "문자열에서 대문자는 소문자로, 소문자는 대문자로 바꿔서 출력하세요."

**[하수]** 반복문 돌리기

```python
res = ""
for c in s:
    if c.isupper(): res += c.lower()
    else: res += c.upper()
```

**[고수]** `swapcase()` 

```python
res = s.swapcase()
```

**[초고수]** `re.sub` 활용 (함수 매핑)

- 복잡한 조건일 때만 사용하세요.

```python
# [초고수] re.sub 활용 (함수 매핑)
# 복잡한 조건(예: 숫자는 냅두고 문자만 뒤집기 등)일 때 사용합니다.

import re

# 정규식으로 모든 알파벳([a-zA-Z])을 잡아서, 람다 함수로 대소문자 로직을 태움
# x.group(0)은 매칭된 문자 하나하나를 뜻함
re.sub(r"[a-zA-Z]", lambda x: x.group(0).lower() if x.group(0).isupper() else x.group(0).upper(), s)
```





