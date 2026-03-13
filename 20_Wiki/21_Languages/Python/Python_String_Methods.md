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
  - chr
  - swapcase
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

# Python_String_Methods

## 개념 한 줄 요약

> **"문자열을 쪼개고, 붙이고, 교체하고, 검증하는 도구 모음."**

|도구|역할|
|---|---|
|`split()`|문자열을 쪼개서 리스트로|
|`join()`|리스트를 문자열로 합치기|
|`replace()`|특정 문자 찾아서 교체|
|`strip()`|양끝 공백·쓰레기 제거|
|`translate()`|여러 문자를 한 번에 교체|
|`count()`|특정 문자·문자열 등장 횟수 세기|
|`isdigit()`|숫자인지 문자인지 확인|
|`ord() / chr()`|문자 ↔ 숫자 변환|
|`swapcase()`|대↔소 뒤집기|


---

---

# ① split() — 쪼개기

> 문자열을 특정 기준(구분자)으로 잘라서 **리스트로 반환**한다.

```python
# 공백 기준 (스페이스·탭·엔터 전부 처리)
text = "Spark is   very    fast"
text.split()
# ['Spark', 'is', 'very', 'fast']  <- 공백 여러 개도 알아서 처리

# 특정 문자 기준
"apple,banana,grape".split(",")
# ['apple', 'banana', 'grape']

# maxsplit: 딱 n번만 자르기
log = "ERROR 2026-01-27 서버가 터졌습니다"
log.split(" ", 1)
# ['ERROR', '2026-01-27 서버가 터졌습니다']  <- 로그 레벨 분리
```

## rsplit() — 오른쪽에서 자르기

```python
# 파일명·확장자 분리
filename = "data.v1.final.csv"
name, ext = filename.rsplit(".", 1)
# name = "data.v1.final"
# ext  = "csv"
# 앞에서 자르면 "data" 만 분리됨 -> 오른쪽에서 잘라야 정확
```

## 빈 문자열 함정

```python
"".split(",")
# ['']  <- 빈 리스트 [] 가 아님!

len("".split(","))  # 1  <- 0이 아니라 1 -> 개수 셀 때 버그 유발
```

---

---

# ② join() — 합치기

> **`"구분자".join(리스트)`** 리스트 요소 사이에 구분자를 끼워 하나의 문자열로 합친다.

```python
words = ['Spark', 'is', 'cool']

" ".join(words)    # 'Spark is cool'
" -> ".join(words) # 'Spark -> is -> cool'
"".join(words)     # 'Sparkiscool'
```

## 이스케이프 문자 구분자

```python
answer = ['apple', 'banana', 'grape']

print('\n'.join(answer))
# apple
# banana
# grape

print('\t'.join(answer))
# apple    banana    grape
```

---

---

# ③ strip() — 공백·쓰레기 제거

> 파일을 읽어오면 눈에 보이지 않는 공백·줄바꿈이 섞여 에러를 유발한다. **split 하기 전에 무조건 strip 습관화.**

```python
text = "   \n\t  Hello, Airflow!  \n  "

text.strip()    # 'Hello, Airflow!'           <- 양쪽 전부
text.lstrip()   # 'Hello, Airflow!  \n  '     <- 왼쪽만
text.rstrip()   # '   \n\t  Hello, Airflow!'  <- 오른쪽만
```

## 특정 문자 제거

```python
dirty = "000000Data_Pipeline_001.csv.bak"
dirty.lstrip("0").rstrip(".bak")
# 'Data_Pipeline_001.csv'
```

## 실무 패턴 — readlines() 줄바꿈 제거

```python
for line in open('file.txt').readlines():
    line = line.rstrip('\n')  # 줄 끝 \n 제거 후 처리
```

---

---

# ④ replace() — 갈아끼우기

> "위치(Index)" 가 아니라 "값(Value)" 을 찾아서 바꾼다.

```python
text = "I like Java. Java is heavy."
text.replace("Java", "Python")
# 'I like Python. Python is heavy.'  <- 전부 다 바뀜

# 삭제: 빈 문자열로 교체
"1,000,000원".replace(",", "").replace("원", "")
# '1000000'  <- int() 변환 가능
```

## 원본은 안 바뀐다

```python
s = "Hello"
s.replace("H", "J")  # 이렇게만 쓰면 아무 일도 안 일어남!
print(s)  # 여전히 'Hello'

# 반드시 변수에 다시 담아야 함
s = s.replace("H", "J")
print(s)  # 'Jello'
```

---

---

# ⑤ translate() + maketrans() — 한 번에 여러 개 바꾸기

> **replace 는 순차 실행이라 A→B, B→A 를 동시에 못 한다.** 변환 테이블을 만들어서 한 번에 적용해야 한다.

## replace 두 번의 함정

```python
text = "AB"
text.replace("A", "B").replace("B", "A")
# 'AA'  <- 원래 의도는 'BA' 였는데!
# A->B 로 먼저 바꿔서 "BB" 가 된 다음, B->A 로 바꿔서 "AA"
```

## translate 로 해결

```python
text = "Hello Python World"
table = str.maketrans("ABo", "BA0")  # 길이 같아야 함
text.translate(table)
# 'Hell0 Pyth0n W0rld'  <- 한 번에 동시 적용
```

## 실전 활용 — 숫자를 문자로 변환

```python
def solution(age):
    aa = str(age)
    return aa.translate(str.maketrans('0123456789', 'abcdefghij'))

solution(25)  # 'cf'  (2->c, 5->f)
```

```
str.maketrans('0123456789', 'abcdefghij')
  0->a, 1->b, 2->c, 3->d, 4->e
  5->f, 6->g, 7->h, 8->i, 9->j
```

|방식|코드|
|---|---|
|`chr` 방식|`"".join(chr(97 + int(i)) for i in str(age))`|
|`translate` 방식|`str(age).translate(str.maketrans('0123456789', 'abcdefghij'))`|

> 둘 다 같은 결과. translate 가 루프 없이 더 간결하다.

---

---

# ⑥ 문자 ↔ 숫자 변환 — ord() / chr()

> **`ord()` : 문자 → 아스키(유니코드) 번호** **`chr()` : 아스키 번호 → 문자**

```python
ord('a')  # 97
ord('A')  # 65
ord('0')  # 48
ord('z')  # 122

chr(97)   # 'a'
chr(65)   # 'A'
chr(48)   # '0'
```

## 아스키 번호 체계

```
숫자:   '0'=48  ~  '9'=57   (총 10개)
대문자: 'A'=65  ~  'Z'=90   (총 26개)
소문자: 'a'=97  ~  'z'=122  (총 26개)

대문자 -> 소문자: +32
```

## 활용 패턴

```python
# 숫자 자리 -> 알파벳 변환 (chr 방식)
"".join(chr(97 + int(i)) for i in "259")
# 'cfj'  (2->c, 5->f, 9->j)

# 알파벳 순서 계산
ord('c') - ord('a')  # 2  <- 'c' 는 0부터 세면 2번째
ord('z') - ord('a')  # 25

# 대문자 -> 소문자 변환
chr(ord('A') + 32)   # 'a'

# 알파벳 순환 (z 다음은 a)
chr((ord('z') - ord('a') + 1) % 26 + ord('a'))  # 'a'
```

## translate vs chr — 언제 뭘 쓰나

```
translate: 변환 규칙이 고정적이고 루프 없이 처리하고 싶을 때
chr/ord:   변환 규칙이 수식으로 표현 가능할 때 (순환, 암호화 등)
```

---

---

# ⑦upper() / lower() / swapcase() — 대소문자 변환

```python
text = "Python Is Easy"
text.upper()     # 'PYTHON IS EASY'   전부 대문자
text.lower()     # 'python is easy'   전부 소문자
text.swapcase()  # 'pYTHON iS eASY'   대↔소 뒤집기

# 대소문자 무시하고 비교
user_input = "Yes"
if user_input.lower() == "yes":
    print("통과!")
```

```text
swapcase():
  대문자 → 소문자
  소문자 → 대문자
  동시에 뒤집음
```

---

---

# ⑧ startswith() / endswith() — 접두사·접미사 검사

```python
filename = "data_2024.csv"

filename.startswith("data_")             # True
filename.endswith(".txt")                # False
filename.endswith((".csv", ".txt"))      # True  <- 튜플로 여러 조건 OR
filename.lower().endswith(".csv")        # True  <- 대소문자 무시
```

## 리스트 컴프리헨션과 결합

```python
files = ["data.csv", "README.md", "data_backup.txt", "result.csv"]

csv_files  = [f for f in files if f.endswith(".csv")]
data_files = [f for f in files if f.startswith("data_")]

count = sum(f.endswith(".csv") for f in files)  # 2
```

## 불리언 → 정수 변환

```python
def check_prefix(my_string, prefix):
    return int(my_string.startswith(prefix))
# True -> 1, False -> 0
```

---
---
# ⑨ count() — 등장 횟수 세기

> 문자열 안에서 특정 문자 또는 문자열이 몇 번 나오는지 센다.


```python
# 문자열에 바로 .count() 가능 — 리스트 변환 불필요
order = "369369"
order.count('3')   # 2
order.count('6')   # 2
order.count('9')   # 2

# 여러 개 동시에 셀 때
order.count('3') + order.count('6') + order.count('9')   # 6
```

## count() vs Counter — 언제 뭘 쓰나

```python
from collections import Counter

text = "369369"

# count() — 특정 문자 1~2개만 셀 때
text.count('3')   # 2  ← 간단, import 불필요

# Counter — 전체 문자 빈도를 한 번에 볼 때
Counter(text)
# Counter({'3': 2, '6': 2, '9': 2})  ← 딕셔너리처럼 사용 가능
Counter(text)['3']   # 2
```

```
count()   특정 문자 몇 개인지만 궁금할 때  → 짧고 간단
Counter   전체 문자 빈도를 한꺼번에 볼 때  → 딕셔너리처럼 사용 가능
          없는 키 조회해도 KeyError 없이 0 반환
```

```python
# Counter 의 추가 기능
c = Counter("aabbbcc")
c.most_common(2)   # [('b', 3), ('a', 2)]  ← 빈도 높은 순 TOP N
```

>더 자세한 내용은 [[Python_Collections_Counter]] 참고 

## 부분 문자열도 셀 수 있음


```python
text = "banana"
text.count("an")   # 2  ('an', 'an')
text.count("na")   # 2  ('na', 'na')

# 단, 겹치는 구간은 세지 않음
"aaa".count("aa")  # 1  ('aa' + 'a', 앞의 aa 소비 후 'a' 만 남는다.)
```

---

---

# ⑩ isdigit() / isalpha() / isalnum() — 데이터 검증

|함수|True 조건|
|---|---|
|`.isdigit()`|`0~9` 만 있을 때|
|`.isalpha()`|한글·영어만 있을 때|
|`.isalnum()`|숫자+문자 조합 (특수문자·공백 없을 때)|

```python
"25".isdigit()     # True
"Alice".isalpha()  # True
"1000원".isdigit() # False  <- '원' 때문에
"-5".isdigit()     # False  <- '-' 때문에
"3.14".isdigit()   # False  <- '.' 때문에
```

## 음수·소수점은 False — float 으로 우회

```python
def is_number(s):
    try:
        float(s)
        return True
    except ValueError:
        return False

is_number("3.14")   # True
is_number("-10")    # True
is_number("1,000")  # False  <- 쉼표 때문에
```

---

---

# 전체 치트시트

|함수|동작|반환|
|---|---|---|
|`split(sep)`|구분자로 쪼개기|리스트|
|`rsplit(sep, n)`|오른쪽에서 n번 쪼개기|리스트|
|`join(list)`|리스트 합치기|문자열|
|`strip()`|양끝 공백 제거|문자열|
|`replace(a, b)`|a 를 b 로 교체 (원본 유지)|문자열|
|`translate(table)`|변환 테이블로 한 번에 교체|문자열|
|`upper() / lower()`|대소문자 변환|문자열|
|`startswith() / endswith()`|접두·접미사 확인|bool|
|`isdigit()`|전부 숫자?|bool|
|`isalpha()`|전부 문자?|bool|
|`ord(c)`|문자 -> 아스키 번호|int|
|`chr(n)`|아스키 번호 -> 문자|str|