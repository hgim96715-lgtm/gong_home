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
## 대소문자 변환 3대장 (Case Conversion) 

`if char.isupper():` 처럼 하나하나 검사하지 마세요. 
통째로 바꾸는 게 빠릅니다.

| 메서드               | 설명                | 예시 (`s = "PyThon"`)    |
| :---------------- | :---------------- | :--------------------- |
| **`.upper()`**    | 싹 다 대문자로          | `"PYTHON"`             |
| **`.lower()`**    | 싹 다 소문자로          | `"python"`             |
| **`.swapcase()`** | **대↔소 뒤집기** (기억!) | `"pYtHON"`             |
| `.capitalize()`   | 문장 맨 앞만 대문자       | `"Python"`             |
| `.title()`        | 단어마다 앞글자 대문자      | `"Python Programming"` |

> **💡 상태 확인용 (True/False)**
> * `.isupper()`, `.islower()`

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





