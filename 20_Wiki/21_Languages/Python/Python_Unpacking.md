---
aliases:
  - Unpacking
  - 언패킹
  - 튜플 풀기
  - 리스트 풀기
  - 구조 분해 할당
  - Asterisk
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
---
## 개념 한 줄 요약 (Concept Summary)

**"리스트나 튜플 등 '묶여있는 데이터'를 개별 변수 여러 개로 한 번에 '풀어서(Unpack)' 담는 문법."**

---
## 기본 문법 (Basic Unpacking)

우변(데이터)의 개수와 좌변(변수)의 개수가 **정확히 일치**해야 합니다.

### ① 리스트/튜플 풀기

```python
# 리스트 풀기
data = [10, 20]
a, b = data
print(a) # 10
print(b) # 20

# 튜플 풀기 (괄호 생략 가능)
name, age = ("Kim", 25)
print(name) # "Kim"
```

### ② 실무 활용: 함수 리턴값 받기

함수가 여러 개의 값을 반환할 때 언패킹이 자동으로 일어납니다.

### ③ 반복문에서 바로 풀기 (List of Lists)

리스트 안에 리스트가 들어있는 경우(`[[A, B], [C, D]]`), `for`문을 돌릴 때 변수를 두 개 지정하면 **꺼냄과 동시에 언패킹**이 됩니다.

```python
# 학생 이름과 점수가 담긴 이중 리스트
scores = [["Kim", 90], ["Lee", 80], ["Park", 95]]

# (Bad) 일단 꺼내고 나서 다시 나누는 방식 (번거로움)
for student in scores:
    name = student[0]
    score = student[1]
    # 혹은 name, score = student (두 줄 써야 함)

# (Good) for문에서 바로 언패킹! (깔끔)
for name, score in scores:
    print(f"{name} 학생: {score}점")
```

**⚠️ 주의 (개수 일치):** 
안쪽 리스트가 2개씩(`[이름, 점수]`) 묶여있다면, `for`문 변수도 반드시 **2개**여야 합니다. 
만약 `for name, score, grade in scores:` 처럼 3개를 받으려고 하면 **ValueError**가 발생합니다.

**⚠️ 내가 헷갈렷던 부분**
`arr = [[a,b], [c,d], [e,f]]` 일 때,
1. **`for q in arr:`** 라고 하면 `q`에는 `[a, b]` 리스트 자체가 하나 들어옵니다.
2. 그래서 `one, two, three = q`를 하셨다면, `q`는 2개인데 변수는 3개라서 에러가 발생

---
## 에러 주의 (ValueError)

개수가 안 맞으면 에러가 터집니다. 초보자가 가장 많이 겪는 에러 중 하나입니다.

```python
data = [1, 2, 3]

# (X) 변수가 부족함 -> ValueError: too many values to unpack
a, b = data 

# (X) 변수가 너무 많음 -> ValueError: not enough values to unpack
a, b, c, d = data
```

---
## 나머지 몽땅 담기 (`*`)

별표(`*`)는 앞, 중간, 뒤 어디든 올 수 있지만, **한 식에 딱 1개만** 쓸 수 있습니다.

### ① `*rest` 활용

데이터 개수가 가변적일 때, **별표(`*`)** 를 쓰면 "나머지 전부"를 리스트로 묶어서 가져옵니다.

```python
numbers = [1, 2, 3, 4, 5]

# 첫 번째만 a에 담고, 나머지는 rest에 담아라
a, *rest = numbers

print(a)    # 1
print(rest) # [2, 3, 4, 5]
```

### ② 앞뒤로 쪼개기

```python
data = [100, 200, 300, 400, 500]

# 처음과 끝만 가져오고 중간은 무시
first, *middle, last = data

print(first) # 100
print(last)  # 500
print(middle) # [200, 300, 400]
```

---
## 필요 없는 값 버리기 (`_`)

변수 이름으로 **언더바(`_`)** 를 쓰면 "이 값은 안 쓸 거니까 버려라(Placeholder)"라는 관습적인 약속입니다.

```python
user_info = ["admin", "1234", "Seoul", "010-1234-5678"]

# ID와 지역만 필요하고, 비밀번호랑 전화번호는 필요 없음
user_id, _, location, _ = user_info

print(user_id)  # "admin"
print(location) # "Seoul"
# _ 변수에는 마지막 값("010...")이 들어있긴 하지만 신경 쓰지 않음
```

---
## 반복문에서의 활용 (For Loop)

`enumerate`나 `zip`을 쓸 때 언패킹은 필수입니다.

```python
pairs = [("A", 1), ("B", 2), ("C", 3)]

# 리스트 안의 튜플을 바로 a, b로 언패킹
for char, num in pairs:
    print(f"{char}는 {num}입니다.")
```

