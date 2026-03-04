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
  - isdigit
  - isalpha
  - translate
  - maketrans
tags:
  - Python
related:
  - "[[Python_Variables_Types]]"
  - "[[Python_Lists_Tuples]]"
  - "[[00_Python_HomePage]]"
  - "[[Spark_General_Transformations]]"
  - "[[Python_String_Indexing_Slicing]]"
  - "[[Python_Membership_In]]"
  - "[[SQL_Regular_Expression]]"
---

# Python 문자열 메서드 (String Methods)

## 개념 한 줄 요약

> **"긴 문장을 쪼개서(split) 리스트로 만들거나, 리스트를 다시 문장으로 붙이는(join) 기술."**

|핵심 도구|역할|
|---|---|
|`split()`|칼 — 문자열을 쪼개서 리스트로|
|`join()`|풀 — 리스트를 문자열로 합치기|
|`replace()`|수정테이프 — 특정 문자 찾아서 교체|
|`strip()`|지우개 — 양끝 공백·쓰레기 제거|
|`isdigit()`|감별사 — 숫자인지 문자인지 확인|

---

---

# ① split() — 쪼개기

> **문자열을 특정 기준(구분자)으로 잘라서 리스트로 반환한다.**

## 기본 사용법 — 공백 기준

괄호 안에 아무것도 안 넣으면 스페이스·탭·엔터를 기준으로 자른다.

```python
text = "Spark is   very    fast"

words = text.split()
print(words)
# ['Spark', 'is', 'very', 'fast']
# 공백이 여러 개여도 알아서 예쁘게 처리 (스마트!)
```

## 특정 문자로 자르기 — CSV 파싱

```python
data = "apple,banana,grape"

fruits = data.split(",")
print(fruits)
# ['apple', 'banana', 'grape']
```

## maxsplit — 딱 n번만 자르기

```python
log = "ERROR 2026-01-27 서버가 터졌습니다 긴급 상황"

parts = log.split(" ", 1)   # 앞에서 딱 1번만 자름
print(parts)
# ['ERROR', '2026-01-27 서버가 터졌습니다 긴급 상황']
# 로그 레벨과 나머지 내용을 분리할 때 유용
```

## rsplit() — 오른쪽에서 자르기 (파일명·확장자 분리)

`data.v1.final.csv` 처럼 점(`.`) 이 여러 개면 앞에서 자르면 엉망이 된다.

```python
filename = "data.v1.final.csv"

name, ext = filename.rsplit(".", 1)   # 오른쪽에서 1번만 자름
print(name)   # data.v1.final
print(ext)    # csv
```

## ⚠️ 빈 문자열 함정

```python
"".split(",")
# 결과: ['']  ← 빈 리스트 [] 가 아님!

len("".split(","))   # 1  ← 0이 아니라 1이 나와서 버그 유발
```

> 빈 문자열에 split 하면 `['']` (빈 문자열이 담긴 리스트) 가 나온다. 개수 셀 때 1개로 잘못 세는 버그가 자주 생기니 주의.

---

---

# ② join() — 합치기

> **문법: `"구분자".join(리스트)`** 리스트의 각 요소 사이에 구분자를 끼워 넣으며 하나의 문자열로 합친다.

## 기본 사용법

```python
words = ['Spark', 'is', 'cool']

# 스페이스로 합치기
sentence = " ".join(words)
print(sentence)
# 'Spark is cool'

# 화살표로 합치기
arrow = " -> ".join(words)
print(arrow)
# 'Spark -> is -> cool'

# 구분자 없이 붙이기
concat = "".join(words)
print(concat)
# 'Sparkiscool'
```

## 이스케이프 문자를 구분자로 — 줄바꿈·탭 삽입

```python
answer = ['apple', 'banana', 'grape']

# 각 항목을 줄바꿈으로 구분해서 출력
print('\n'.join(answer))
# apple
# banana
# grape

# 탭으로 구분
print('\t'.join(answer))
# apple    banana    grape

# 실전: 여러 줄 텍스트를 파일로 저장할 때
lines = ['1번 줄', '2번 줄', '3번 줄']
content = '\n'.join(lines)
with open('output.txt', 'w') as f:
    f.write(content)
```

## Spark 활용

```python
# RDD: 문장을 단어 단위로 쪼개기
rdd.flatMap(lambda line: line.split(" "))

# "Hello World" → split → ["Hello", "World"] → flatMap → Hello, World (낱개)
```

---

---

# ③ strip() / lstrip() / rstrip() — 공백·쓰레기 제거

> **파일을 읽어오면 눈에 보이지 않는 공백·줄바꿈이 섞여 에러를 유발한다.** split 하기 전에 무조건 strip 을 습관화한다.

## 기본 사용법 — 공백·탭·줄바꿈 제거

```python
text = "   \n\t  Hello, Airflow!  \n  "

print(text.strip())    # 'Hello, Airflow!'   ← 양쪽 전부 제거
print(text.lstrip())   # 'Hello, Airflow!  \n  '  ← 왼쪽만
print(text.rstrip())   # '   \n\t  Hello, Airflow!'  ← 오른쪽만
```

## 특정 문자 타겟팅 — 양끝에서 지정 문자 깎아내기

```python
dirty = "000000Data_Pipeline_001.csv.bak"

clean = dirty.lstrip("0").rstrip(".bak")
print(clean)
# 'Data_Pipeline_001.csv'
```

> **실무 필수 패턴:** 텍스트 파일을 `readlines()` 로 읽으면 줄 끝에 `\n` 이 붙는다.

```python
 for line in open('file.txt').readlines():
     line = line.rstrip('\n')   # 줄바꿈 제거 후 처리
```

---

---

# ④ replace() — 갈아끼우기

> **"위치(Index)" 가 아니라 "값(Value)" 을 찾아서 바꾼다.**

## 기본 사용법

```python
text = "I like Java. Java is heavy."

new_text = text.replace("Java", "Python")
print(new_text)
# 'I like Python. Python is heavy.'  (모두 다 바뀜)
```

## 삭제 — 빈 문자열로 교체

```python
money = "1,000,000원"

clean = money.replace(",", "").replace("원", "")
print(clean)
# '1000000'  ← 이제 int() 변환 가능
```

## ⚠️ 원본은 안 바뀐다 — 반드시 변수에 다시 담아야 함

```python
s = "Hello"
s.replace("H", "J")   # 이렇게만 쓰면 아무 일도 안 일어남!

print(s)   # 여전히 'Hello'

# ✅ 변수에 다시 담아야 함
s = s.replace("H", "J")
print(s)   # 'Jello'
```

---

---

# ⑤ translate() + maketrans() — 한 번에 여러 개 바꾸기

> **replace 는 순차 실행이라 A→B, B→A 를 동시에 못 한다.** 변환 테이블을 만들어서 한 번에 적용해야 한다.

## replace 두 번의 함정

```python
text = "AB"

# A → B 로 바뀜 ("BB") → 그 다음 B → A 로 바뀜 ("AA")
broken = text.replace("A", "B").replace("B", "A")
print(broken)   # 'AA'  (원래 의도는 'BA' 였는데!)
```

## translate 로 해결 — 한 번에 적용

```python
text = "Hello Python World"

# maketrans("찾을거", "바꿀거") — 길이가 같아야 함
table = str.maketrans("ABo", "BA0")

result = text.translate(table)
print(result)
# 'Hell0 Pyth0n W0rld'

# 실전: 암호화 (Caesar Cipher)
code = str.maketrans("abcde", "12345")
print("apple".translate(code))   # '1ppl5'
```

|함수|용도|
|---|---|
|`replace()`|단어(문자열) 를 통째로 교체|
|`translate()`|글자(Character) 를 1:1 매핑으로 교체|

---

---

# ⑥ upper() / lower() — 대소문자 변환

> **`lower(변수)` ❌ → `변수.lower()` ✅** 파이썬 문자열 함수는 메서드라서 변수 뒤에 점을 찍어야 한다.

```python
text = "Python Is Easy"

print(text.upper())   # 'PYTHON IS EASY'
print(text.lower())   # 'python is easy'
```

## 실무 활용 — 대소문자 무시하고 비교 (정규화)

```python
user_input = "Yes"

# ❌ 그냥 비교: 'Yes' != 'yes' → 조건 불만족
if user_input == "yes":
    print("통과")   # 실행 안 됨

# ✅ 소문자 변환 후 비교
if user_input.lower() == "yes":
    print("통과!")  # 실행 됨
```

---

---

# ⑦ startswith() / endswith() — 접두사·접미사 검사

> **파일 목록에서 원하는 것만 필터링할 때 가장 많이 쓴다.**

## 기본 사용법

```python
filename = "data_2024.csv"

print(filename.startswith("data_"))   # True
print(filename.endswith(".txt"))      # False
```

## 리스트 컴프리헨션과 결합

```python
files = ["data.csv", "README.md", "data_backup.txt", "result.csv"]

csv_files  = [f for f in files if f.endswith(".csv")]
data_files = [f for f in files if f.startswith("data_")]
```

## 튜플로 여러 조건 한 번에 — OR 처럼 동작

```python
# .jpg 또는 .png 로 끝나는지 확인
if file.endswith((".jpg", ".png")):
    pass
```

## 대소문자 무시하고 검사

```python
# "Data.CSV" 처럼 대소문자 섞인 경우
if file.lower().endswith(".csv"):
    pass
```

## 불리언 → 정수 변환 (조건문 없애기)

```python
# True=1, False=0 이므로 int() 로 바로 변환 가능
def check_prefix(my_string, prefix):
    return int(my_string.startswith(prefix))

# 리스트에서 조건 만족 개수 세기
count = sum(f.endswith(".csv") for f in files)
```

---

---

# ⑧ isdigit() / isalpha() / isalnum() — 데이터 검증

> **사용자 입력이나 파일 데이터가 숫자인지 문자인지 검사할 때 사용.**

## 주요 함수 3대장

|함수|질문|True 조건|
|---|---|---|
|`.isdigit()`|전부 숫자니?|`0~9` 만 있을 때|
|`.isalpha()`|전부 문자니?|한글·영어만 있을 때|
|`.isalnum()`|숫자+문자 조합이니?|특수문자·공백 없을 때|

```python
print("25".isdigit())     # True
print("Alice".isalpha())  # True
print("1000원".isdigit()) # False  ← '원' 때문에
print("-5".isdigit())     # False  ← '-' 때문에
print("3.14".isdigit())   # False  ← '.' 때문에
```

## ⚠️ 음수·소수점은 False — float 변환으로 우회

```python
def is_number(s):
    try:
        float(s)    # 변환 시도
        return True
    except ValueError:
        return False

print(is_number("3.14"))   # True
print(is_number("-10"))    # True
print(is_number("1,000"))  # False  ← 쉼표 때문에
```

---

---
