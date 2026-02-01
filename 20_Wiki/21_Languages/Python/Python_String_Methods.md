---
aliases:
  - split
  - join
  - strip
  - replace
  - 문자열함수
  - 파싱
  - Parsing
  - 확장자 추출
tags:
  - Python
related:
  - "[[Python_Variables_Types]]"
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
  - "[[Spark_General_Transformations]]"
  - "[[Python_String_Indexing_Slicing]]"
---
## 개념 한 줄 요약

**"긴 문장을 쪼개서(Split) 리스트로 만들거나, 리스트를 다시 문장으로 붙이는(Join) 기술."**

- **데이터 엔지니어링의 일상:** "콤마(`,`)로 구분된 CSV 파일을 읽어서 리스트로 바꿔라."
- **핵심 도구:** `split` (칼), `join` (풀), `replace` (수정테이프)

---
## 쪼개기: `split()` 

문자열을 특정 기준(구분자)으로 잘라서 **리스트(List)** 로 반환합니다.

### ① 기본 사용법 (공백 기준)

괄호 안에 아무것도 안 넣으면 **스페이스, 탭, 엔터**를 기준으로 자릅니다.

```python
text = "Spark is   very    fast"

# 공백이 여러 개여도 알아서 예쁘게 잘라줌 (스마트함!)
words = text.split()
print(words)
# 결과: ['Spark', 'is', 'very', 'fast']
```

### ② 특정 문자로 자르기 (CSV 파싱)

```python
data = "apple,banana,grape"

# 쉼표(,)를 기준으로 자름
fruits = data.split(",")
print(fruits)
# 결과: ['apple', 'banana', 'grape']
```

### ③ 딱 n번만 자르기 (`maxsplit`)

```python
log = "ERROR 2026-01-27 서버가 터졌습니다 긴급 상황"

# 앞에서부터 딱 1번만 자르고, 나머지는 덩어리로 둠
parts = log.split(" ", 1)
print(parts)
# 결과: ['ERROR', '2026-01-27 서버가 터졌습니다 긴급 상황']
```

---
## 반대 기술: `join()`

은근히 모르는 꿀기능입니다. 
`split`으로 쪼갠 리스트를 다시 **문자열 하나로 합칠 때** 씁니다.

- `{python}문법: "구분자".join(리스트)`

```python
words = ['Spark', 'is', 'cool']

# 문법: "구분자".join(리스트)
# "스페이스를 사이에 끼워넣으면서 합쳐라"
sentence = " ".join(words)
print(sentence)
# 결과: "Spark is cool"

# "화살표를 끼워넣어라"
arrow = " -> ".join(words)
print(arrow)
# 결과: "Spark -> is -> cool"
```

---
## 청소하기: `strip()` 

사용자 입력이나 파일 읽을 때 **앞뒤 공백(엔터 포함)** 을 제거합니다.
`split` 하기 전에 무조건 해주는 게 좋습니다.

```python
raw_data = "  Hello Python  \n"

clean_data = raw_data.strip()
print(clean_data)
# 결과: "Hello Python" (깔끔!)
```

---
##  갈아끼우기: `replace()` 

특정 글자를 찾아서 다른 글자로 바꿔줍니다. 
오타 수정이나 불필요한 기호 삭제할 때 씁니다.

>**"위치(Index)"가 아니라 "값(Value)"** 을 찾아서 바꾼다!!!!!!

### ① 기본 사용법

```python
text = "I like Java. Java is heavy."

# "Java"를 찾아서 "Python"으로 바꿔라
new_text = text.replace("Java", "Python")

print(new_text)
# 결과: "I like Python. Python is heavy." (모두 다 바뀜)
```

### ② 글자 삭제하기

"바꿀 글자"에 **빈 문자열(`""`)** 을 넣으면 삭제 효과가 납니다.

```python
money = "1,000,000원"

# 쉼표(,)랑 "원"을 없애고 싶을 때
clean_money = money.replace(",", "").replace("원", "")

print(clean_money)
# 결과: "1000000" (이제 숫자로 변환 가능!)
```

### ⚠️ 주의: 원본은 안 바뀐다! (Immutability)

파이썬의 문자열은 **'불변(Immutable)'** 입니다. 
`replace`를 했다고 원본 변수가 바뀌지 않습니다. 반드시 변수에 다시 담아줘야 합니다.

```python
s = "Hello"
s.replace("H", "J") # 이렇게만 쓰면 아무 일도 안 일어난 것처럼 보임!

print(s) # 여전히 "Hello"

# (O) 이렇게 변수에 담아야 함
s = s.replace("H", "J") 
print(s) # "Jello"
```

---
## [실전 패턴] 파일명 다루기 (확장자 추출) 

데이터 엔지니어링에서 파일 목록(`list`)을 필터링하거나 이름을 바꿀 때 무조건 쓰는 패턴입니다.

### 1. 특정 파일만 골라내기 (`endswith`, `startswith`)

`split`으로 쪼개서 비교하지 마세요! 가독성이 떨어집니다.

**Situation:** `.csv` 파일만 리스트에 담고 싶다.

```python
files = ["data.csv", "README.md", "test.py", "result.csv"]

# [Bad] 쪼개서 확인하기 (복잡함)
# if file.split(".")[-1] == 'csv': ...

# [Good] 직관적인 함수 사용 ✨
csv_files = [f for f in files if f.endswith(".csv")]
print(csv_files) 
# 결과: ['data.csv', 'result.csv']
```

>**꿀팁:** 여러 개를 동시에 검사할 수도 있습니다. 
>`if file.endswith((".csv", ".xls", ".json")): ...` (튜플로 넣기)`startswith / endswith`에 여러 조건 → 무조건 튜플로!
> **꿀팁:** 시작하는 문자 찾기는 **`startswith("data")`** 를 쓰면 됩니다.

### ② 파일명과 확장자 분리 (`rsplit`)

`data.v1.final.csv` 처럼 점(`.`)이 많은 파일은 앞에서 자르면 엉망이 됩니다. 
**오른쪽(Right) 끝에서** 잘라야 합니다.

```python
filename = "data.v1.final.csv"

# rsplit(".", 1) -> 오른쪽에서 점을 기준으로 딱 1번만 잘라라
name, ext = filename.rsplit(".", 1)

print(name) # data.v1.final
print(ext)  # csv
```

---
## Spark에서의 활용 (방금 본 코드 복습)

```python
# RDD Transformation
# "문장 하나(line)를 받아서 스페이스(" ") 기준으로 쪼개라"
rdd.flatMap(lambda line: line.split(" "))
```

- **데이터:** `"Hello World"` (String)
- **`split(" ")` 적용 후:** `["Hello", "World"]` (List)
- **`flatMap` 적용:** 리스트가 벗겨지고 `Hello`, `World`가 낱개로 퍼짐.

---
### 주의사항 (버그 예방)

**1. 빈 문자열의 함정** "데이터 파일(CSV, 로그)을 다룰 때 `text.split(",")`을 많이 쓰는데, 만약 `text`가 텅 빈 문자열(`""`)이라면? 빈 리스트(`[]`)가 나올 것 같지? 아니야! **`['']` (빈 문자열이 하나 들어있는 리스트)** 가 나와. 이것 때문에 개수 셀 때 1개라고 잘못 세는 버그가 진짜 많이 생기니까 조심해!"

**2. 변수 재할당 필수** "초보자들이 제일 많이 하는 실수 1위! `text.replace('a', 'b')` 라고만 써놓고 **'어? 왜 안 바뀌지?'** 하고 밤새 고민해. 문자열 함수들은 원본을 건드리지 못해. 항상 **`변수 = 변수.함수()`** 형태로 써야 한다는 거 잊지 마!"

