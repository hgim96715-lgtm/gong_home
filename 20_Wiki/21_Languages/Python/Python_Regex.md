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
  - "[[SQL_Regular_Expression]]"
---
# Python_Regex — 정규표현식

## 한 줄 요약

```
특정 패턴을 가진 문자열을 찾거나 / 바꾸거나 / 쪼개는 도구
import re 로 사용
```

---

---

# ① r"" — Raw String (무조건 붙이기)

```
정규식에는 \d \w \s 처럼 백슬래시가 많음
파이썬에서 \ 는 탈출문자 → 충돌 발생

r"" (Raw String) = "백슬래시를 건드리지 말고 그대로 둬"
```

```python
# r 없이 → 백슬래시 두 번 써야 함
pattern = "\\d{3}-\\d{4}-\\d{4}"   # 가독성 꽝

# r 붙이기 → 그냥 씀
pattern = r"\d{3}-\d{4}-\d{4}"     # 깔끔 ✅
```

```
정규식 쓸 때는 무조건 r"..." 붙이는 습관
```

---

---

# ② 함수 3가지 — 반환값이 전부 다름

|함수|반환값|특징|
|---|---|---|
|`re.search(패턴, 문자열)`|Match Object (1개)|어디서든 첫 번째 1개|
|`re.match(패턴, 문자열)`|Match Object (1개)|문자열 맨 앞에서만|
|`re.findall(패턴, 문자열)`|**리스트**|모두 찾아서 리스트로|
|`re.sub(패턴, 바꿀값, 문자열)`|**str**|찾아서 치환 후 문자열 반환|

---

---

# ③ re.search() — 어디서든 첫 번째 1개

## 반환값: Match Object

```python
import re

text = "제 번호는 010-1234-5678 입니다."
pattern = r"\d{3}-\d{4}-\d{4}"

result = re.search(pattern, text)

print(result)
# <re.Match object; span=(5, 18), match='010-1234-5678'>
# → Match Object (박스) 가 반환됨 / 문자열이 아님!
```

## Match Object 에서 값 꺼내기 — .group()

```python
# 찾은 전체 문자열
result.group()    # '010-1234-5678'
result.group(0)   # '010-1234-5678'  (group() 과 동일)

# 위치 확인
result.span()     # (5, 18)  → (시작인덱스, 끝인덱스)
result.start()    # 5
result.end()      # 18
```

## 반드시 None 체크 후 사용

```python
result = re.search(pattern, text)

# ❌ 바로 .group() 하면 에러 가능
result.group()   # 못 찾으면 AttributeError: 'NoneType'

# ✅ None 체크 먼저
if result:
    print(result.group())
else:
    print("못 찾음")
```

## 그룹핑 () — 일부만 꺼내기

```python
# 괄호로 그룹 지정
m = re.search(r"(\d{3})-(\d{4}-\d{4})", "010-1234-5678")

m.group(0)   # '010-1234-5678'  ← 전체
m.group(1)   # '010'            ← 첫 번째 괄호
m.group(2)   # '1234-5678'      ← 두 번째 괄호
```

```python
# 실전: 로그에서 에러 코드만 추출
log = "[2024-01-30] ERROR: Connection Refused (Code: 503)"

m = re.search(r"Code: (\d+)", log)
if m:
    print(m.group(1))   # '503'  ← 괄호 안만 꺼냄
```

## re.match() — 맨 앞에서만 (주의)

```python
text = "제 번호는 010-1234-5678 입니다."

re.match(r"\d{3}-\d{4}-\d{4}", text)
# → None  ← 맨 앞이 "제" 라서 못 찾음

# 거의 안 씀 → search 쓰는 게 안전
```

---

---

# ④ re.findall() — 전부 찾아서 리스트로

## 반환값: 리스트 (바로 씀 / .group() 필요 없음)

```python
text = "010-1234-5678 이고 집 전화는 02-123-4567 입니다."
pattern = r"\d{2,3}-\d{3,4}-\d{4}"

result = re.findall(pattern, text)

print(result)
# ['010-1234-5678', '02-123-4567']  ← 리스트 바로 반환

# 바로 for 문 사용 가능 (.group() 없음)
for phone in result:
    print(phone)
```

## 그룹 있으면 → 그룹 내용의 리스트

```python
text = "이름: 홍길동, 나이: 30, 이름: 김철수, 나이: 25"

# 그룹 없음 → 전체 매칭 리스트
re.findall(r"\d+", text)
# ['30', '25']

# 그룹 있음 → 그룹 내용 리스트 (튜플)
re.findall(r"이름: (\S+), 나이: (\d+)", text)
# [('홍길동', '30'), ('김철수', '25')]

# 그룹 1개면 → 문자열 리스트
re.findall(r"이름: (\S+)", text)
# ['홍길동', '김철수']
```

```
findall 반환 정리:
  그룹 없음       → ['매칭1', '매칭2']
  그룹 1개        → ['그룹1값', '그룹1값']
  그룹 2개 이상   → [('그룹1', '그룹2'), ...]  튜플 리스트
```

---

---

# ⑤ re.sub() — 찾아서 치환

## 반환값: 문자열 (str)

```python
import re

# 전화번호 뒷자리 마스킹
text = "010-1234-5678"
result = re.sub(r"-\d{4}$", "-****", text)
print(result)   # '010-1234-****'
print(type(result))   # <class 'str'>
```

```python
# 특수문자 제거 (한글 + 숫자 + 공백만 남기기)
text = "안녕하세요!!! 010-1234-5678 ^^"
result = re.sub(r"[^가-힣0-9\s]", "", text)
print(result)   # '안녕하세요 01012345678 '
```

## lambda 활용 — 조건에 따라 다르게 치환

```python
# re.sub 두 번째 인자에 함수 넣기
# 찾은 결과(Match Object) 를 받아서 가공 후 반환

text = "price: 100, discount: 20"

# 숫자를 2배로 바꾸기
result = re.sub(
    r"\d+",
    lambda m: str(int(m.group()) * 2),  # m = Match Object
    text
)
print(result)   # 'price: 200, discount: 40'
```

---

---

# ⑥ span() — 위치 확인

```python
text = "Hello Python World"
m = re.search("Python", text)

m.span()    # (6, 12)  → (시작, 끝)  끝은 포함 안 함
m.start()   # 6
m.end()     # 12

# 좌표로 슬라이싱
start, end = m.span()
text[start:end]   # 'Python'
```

---

---

# 패턴 치트시트

|기호|의미|예시|
|---|---|---|
|`\d`|숫자|`\d{3}` 숫자 3개|
|`\w`|문자+숫자+_|`\w+` 단어 1개 이상|
|`\s`|공백|`\s+` 공백 1개 이상|
|`.`|아무 글자 1개|`a.b` → axb, ayb|
|`*`|0개 이상|`a*`|
|`+`|1개 이상|`\d+`|
|`?`|있거나 없거나|`https?`|
|`^`|문자열 시작|`^\d` 숫자로 시작|
|`$`|문자열 끝|`\d$` 숫자로 끝|
|`[abc]`|a 또는 b 또는 c|`[0-9]`|
|`[^abc]`|a,b,c 제외 전부|`[^가-힣]` 한글 제외|
|`{n}`|정확히 n번|`\d{4}`|
|`{n,m}`|n~m번|`\d{2,4}`|
|`(abc)`|그룹|`(\d+)`|

---

---

# 자주 하는 실수

```python
# ① .group() 전에 None 체크 안 함
m = re.search(r"\d+", "abc")
m.group()   # AttributeError: 'NoneType' → if m: 먼저 체크

# ② findall 에 .group() 쓰려 함
result = re.findall(r"\d+", text)
result.group()   # AttributeError: list has no .group()
# findall 은 이미 리스트 → for 문으로 바로 사용

# ③ r"" 안 붙임
re.search("\d+", text)   # \d 를 탈출문자로 해석 → 오동작
re.search(r"\d+", text)  # ✅

# ④ match 로 중간값 찾으려 함
re.match(r"\d+", "abc123")  # None → search 써야 함
```

---

---

# 함수 반환값 한눈에

```
re.search()   Match Object   → .group() 으로 꺼냄 / None 체크 필수
re.match()    Match Object   → .group() 으로 꺼냄 / 맨 앞만 찾음
re.findall()  list           → 바로 for 문 / .group() 없음
re.sub()      str            → 바로 사용 가능
```