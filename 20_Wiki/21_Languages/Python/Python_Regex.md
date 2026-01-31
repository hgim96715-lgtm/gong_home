---
aliases:
  - 정규표현식
  - regex
  - re
  - re.search
  - re.match
  - re.findall
  - group
  - span
tags:
  - Python
related:
  - "[[Python_String_Methods]]"
  - "[[Python_String_Case_Replace]]"
  - "[[00_Python_HomePage]]"
---
##  개념 한 줄 요약 

**"특정한 규칙(패턴)을 가진 문자열을 찾거나(`search`), 바꾸거나(`sub`), 쪼갤 때(`split`) 쓰는 도구."**

* **핵심:** 파이썬 내장 모듈인 **`import re`** 를 반드시 선언해야 합니다.
* **언제 써요?** "이메일 주소만 뽑아줘", "전화번호 형식이 맞는지 확인해줘", "로그에서 에러 코드만 가져와" 등.

---
## `r` (Raw String)의 마법 

**"백슬래시(`\`)를 두 번(`\\`) 쓰기 귀찮다면, 앞에 `r`을 붙이세요!"**

정규식에는 `\d`, `\w` 처럼 백슬래시가 많이 들어갑니다.
하지만 파이썬에서 `\`는 원래 "탈출 문자(Escape)"라서 충돌이 일어납니다.

###  `r` 없이 쓸 때 (Bad)

백슬래시를 문자로 인식시키려고 **두 번씩** 써야 합니다. (가독성 꽝 )

```python
# 파이썬: "\\d가 뭐야? 아, \d를 쓰고 싶은 거구나."
pattern = "\\d{3}-\\d{4}-\\d{4}"
```

###  `r` 붙여 쓸 때 (Good)

**Raw String(`r"..."`)** 은 "안에 있는 백슬래시를 건드리지 말고 **날것 그대로(Raw)** 둬!" 라는 뜻입니다.

```python
# 파이썬: "오케이, 안에 있는 \d는 건드리지 않을게."
pattern = r"\d{3}-\d{4}-\d{4}"
```

**💡 결론:** 정규식 패턴을 만들 때는 무조건 습관적으로 **`r"..."`** 을 붙이세요!

>"나중에 경로(Path) 다룰 때도 유용해. 
>윈도우 파일 경로가 `C:\User\Name` 이런 식인데, 이것도 `r'C:\User\Name'` 이렇게 쓰면 에러 없이 깔끔하게 들어가.
> **`re` 모듈 쓸 때는 그냥 무조건 `r` 붙인다**고 생각하면 편해!"


---
## 찾는 함수 3대장 (`match` vs `search` vs `findall`) 

가장 많이 헷갈리는 부분입니다. **`search`** 가 가장 범용적입니다.

| 함수 | 설명 | 반환 값 | 특징 |
| :--- | :--- | :--- | :--- |
| **`re.search(패턴, 문자열)`** | **어디서든** 찾음 | Match Object | **가장 추천!** 문장 중간에 있어도 찾음. |
| `re.match(패턴, 문자열)` | **맨 앞부터** 찾음 | Match Object | 문장 중간에 있으면 못 찾음(None). |
| `re.findall(패턴, 문자열)` | **싹 다** 찾음 | **List** `['a', 'b']` | 결과가 리스트로 바로 나옴. (group 필요 없음) |

### ① `re.search()` 사용법 (강추) 

문자열 전체를 훑어서 **가장 먼저 발견되는 하나**를 찾습니다.

```python
import re

text = "제 번호는 010-1234-5678 입니다."

# [패턴] 숫자3개-숫자4개-숫자4개
pattern = r"\d{3}-\d{4}-\d{4}"

result = re.search(pattern, text)

if result:
    print("찾았다!", result.group())  # '010-1234-5678'
else:
    print("못 찾음")
```

### ② `re.match()`의 함정 

`match`는 **"문자열의 시작(Start)"** 부터 패턴이 일치해야 합니다.

```python
text = "제 번호는 010-1234-5678 입니다."
pattern = r"\d{3}-\d{4}-\d{4}"

# 맨 앞글자는 '제'이지 숫자가 아님 -> None 반환
print(re.match(pattern, text)) 
# 결과: None (못 찾음!)
```

----
## 바꾸기: `re.sub()` 

**"찾아서(`search`) + 바꾼다(`replace`)"** 를 한 방에 처리합니다.

데이터 정제(Cleaning)할 때 가장 많이 씁니다.

* **문법:** `{python}re.sub(찾을패턴, 바꿀문자, 원본텍스트)`

### ① 특수문자 싹 다 지우기 (Data Cleaning)

"한글과 숫자만 남기고 나머지는 다 지워줘!"

```python
import re

text = "안녕하세요!!! 010-1234-5678 ^^;;"

# [패턴] 한글(가-힣), 숫자(0-9), 공백(\s)이 "아닌(^)" 것만 찾아라
# ^는 대괄호[] 안에서는 "Not"을 의미함
pattern = r"[^가-힣0-9\s]" 

clean_text = re.sub(pattern, "", text)
print(clean_text) 
# 결과: "안녕하세요 01012345678 " (특수문자 사라짐)
```

### ② 개인정보 마스킹 (Masking)

"전화번호 뒷자리를 `****`로 가려줘!"

```python
text = "010-1234-5678"

# [패턴] -뒤에 숫자4개
pattern = r"-\d{4}$" 

# 해당 패턴을 "-****"로 치환
masked_text = re.sub(pattern, "-****", text)
print(masked_text) 
# 결과: "010-1234-****"
```

----
##  결과 꺼내기 : `group()`과 Match Object 이해하기 

정규식 함수(`re.search`, `re.sub`의 람다 등)는 결과를 바로 **문자열(String)** 로 주지 않습니다.
**`Match Object`** 라는 **"포장 박스"** 에 담아서 줍니다.

### ① 박스(`Match Object`) vs 내용물(`String`)

```python
import re

text = "My number is 010-1234-5678"
pattern = r"\d{3}-\d{4}-\d{4}"  # 전화번호 패턴

# 1. 찾기 (Search)
match = re.search(pattern, text)

# 2. 그냥 출력하면? (박스 채로 확인)
print(match)
# 결과: <re.Match object; span=(13, 26), match='010-1234-5678'>
# 👉 "아, 뭔가 찾긴 찾았는데 객체(Object)네?"

# 3. .group()으로 꺼내기 (내용물 확인)
print(match.group()) 
# 결과: '010-1234-5678'
# 👉 "이제야 진짜 문자열이 나왔네!"
```

### ② 왜 `group(0)`, `group(1)`로 나뉘나요?

정규식에서 **괄호 `()`** 를 쓰면 그룹을 나눌 수 있기 때문입니다.

- **`.group(0)` (또는 `.group()`):** 매칭된 **전체** 문자열 (기본값)
- **`.group(1)`:** 첫 번째 괄호 `()` 안의 내용
- **`.group(2)`:** 두 번째 괄호 `()` 안의 내용

```python
# 전화번호를 국번(010)과 나머지로 쪼개서 찾기
m = re.search(r"(\d{3})-(\d{4}-\d{4})", "010-1234-5678")

print(m.group(0)) # '010-1234-5678' (전체)
print(m.group(1)) # '010'           (첫 번째 괄호)
print(m.group(2)) # '1234-5678'     (두 번째 괄호)
```

**💡 아까 `lambda x: x.group(0)`은 무슨 뜻인가요?** 
`re.sub`이 찾은 건 **Match Object(박스)** 인 `x`입니다. 
람다 함수 안에서 `.lower()` 같은 문자열 함수를 쓰려면, 먼저 `x.group(0)`으로 **박스를 까서 문자열을 꺼내야** 했기 때문입니다!

---
##  위치 확인: `span()` 📍

`match` 객체를 출력해보면 `span=(6, 12)` 같은 숫자가 보입니다.
이건 **"찾은 문자열이 원본의 몇 번째 인덱스에 있는지"** 알려주는 좌표입니다.

* **`span()` 반환값:** `{python}(시작_인덱스, 끝_인덱스)` 튜플 형태
* **주의:** 파이썬 슬라이싱 규칙과 똑같이 **"끝 인덱스는 포함하지 않습니다."**

```python
import re

text = "Hello Python World"
#       0123456789... (인덱스)

match = re.search("Python", text)

# 1. 좌표 확인
print(match.span())  
# 결과: (6, 12) -> 6번째부터 11번째 글자까지라는 뜻

# 2. 좌표로 원본 자르기 (Slicing)
start, end = match.span()
print(text[start:end]) 
# 결과: "Python" (정확히 그 단어가 나옴)
```

- `match.start()` : 시작 인덱스 
- `match.end()` : 끝 인덱스 

---
## 자주 쓰는 패턴 (Cheatsheet) 

| **기호** | **의미**       | **예시**                    |
| ------ | ------------ | ------------------------- |
| `\d`   | 숫자 (Digit)   | `\d{3}` (숫자 3개)           |
| `\w`   | 문자+숫자 (Word) | `\w+` (단어 1개 이상)          |
| `.`    | 아무 글자나 하나    | `.`                       |
| `*`    | 0개 이상 반복     | `a*` (없거나 a, aa, aaa...)  |
| `+`    | **1개 이상 반복** | `\d+` (숫자가 1개 이상 연속됨)     |
| `?`    | 있거나 없거나      | `https?` (http 또는 https)  |
| `^`    | 문자열의 시작      | `^Hello` (Hello로 시작하는 문장) |
| `$`    | 문자열의 끝       | `End$` (End로 끝나는 문장)      |


---
## 실전 예제 (Data Engineering) 

**로그 파일에서 에러 코드만 뽑고 싶다면?**

```python
log = "[2024-01-30 12:00:00] ERROR: Connection Refused (Code: 503)"

# 'Code: ' 뒤에 있는 숫자(\d+)를 찾아라
match = re.search(r"Code: (\d+)", log)

if match:
    error_code = match.group(1) # 괄호 안의 내용만 추출
    print(f"에러 코드: {error_code}") # 503
```



