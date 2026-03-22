---
aliases:
  - 제어문
  - 조건문
  - 반복문
  - If
  - For
  - While
  - 흐름 제어
  - 월러스 연산자
tags:
  - Python
related:
  - "[[00_Python_HomePage]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Python_Lambda_Map]]"
  - "[[Python_Sorting_Logic]]"
---
# Python_Control_Flow — 제어문

## 한 줄 요약

```
코드의 실행 순서를 지휘하는 도구
위에서 아래로만 흐르는 코드를
갈림길(if) 로 나누거나 반복(for/while) 시키기
```

---

---

# ① if — 조건문

```python
if 조건A:
    # A 참일 때
elif 조건B:
    # A 아니고 B 참일 때
else:
    # 전부 거짓일 때
```

```python
status = "failed"

if status == "success":
    print("성공!")
elif status == "failed":
    print("실패! 알림 보내.")
else:
    print("대기 중...")
```

## Truthy / Falsy

```
0         → False
0이 아닌 수 → True  (1, -1, 100 전부)
빈 문자열  → False
빈 리스트  → False
None       → False
```

```python
n = 3
if n % 2:     # 나머지가 1 → True → 홀수
    print("홀수")
else:         # 나머지가 0 → False → 짝수
    print("짝수")
```

---

---

# ② for — 순회 반복

```python
for 변수 in 반복가능한_객체:
    # 꺼낸 값을 처리
```

```python
files = ["data_v1.csv", "data_v2.csv", "data_v3.csv"]

for file in files:
    print(f"Processing {file}...")
```

## for 에 들어올 수 있는 것 (Iterable)

```
✅ 가능:  리스트 [] / 튜플 () / 문자열 "" / 딕셔너리 {} / range()
❌ 불가능: 정수 10 / 실수 3.14  → TypeError

문자열도 Iterable:
  for c in "hello":  →  h e l l o 하나씩 꺼냄
  for c in str(123): →  1 2 3 하나씩 꺼냄  ← list() 변환 불필요
```

## ⭐️ list(str(n)) 안 해도 되는 이유

```python
n = 12345

# ❌ 굳이 list 로 변환 안 해도 됨
for c in list(str(n)):   # list 변환 불필요
    print(c)

# ✅ str 은 그 자체로 Iterable
for c in str(n):         # 바로 순회 가능
    print(c)             # 1 2 3 4 5

# ✅ 한 줄 - 각 자리 숫자 합계
total = sum(int(c) for c in str(n))    # 15
total = sum(int(i) for i in str(n))    # 동일, 변수명만 다름

# ❌ list 변환 후 계산 (불필요한 단계)
total = sum(int(c) for c in list(str(n)))
```

```
list(str(n)) 이 필요한 경우:
  특정 인덱스의 문자를 수정할 때
  "12345"[2] = "9"  ← 문자열은 불변이라 에러
  lst = list("12345"); lst[2] = "9"  ← list 로 변환 후 수정

list(str(n)) 불필요한 경우:
  단순 순회 / 각 글자 처리 / 합계 계산
  → str(n) 만으로 for 에 바로 사용 가능
```

## 제너레이터 표현식 — 괄호 안 for

```python
# 리스트 컴프리헨션 → 리스트 생성
[int(c) for c in str(n)]         # [1, 2, 3, 4, 5]

# 제너레이터 표현식 → sum/max/min 등 집계 함수에 바로 전달
sum(int(c) for c in str(n))      # 15  ← list 안 만들고 바로 합산
max(int(c) for c in str(n))      # 5
min(int(c) for c in str(n))      # 1

# 조건 추가
sum(int(c) for c in str(n) if int(c) % 2 == 0)  # 짝수 자리만 합산
```

```
sum([int(c) for c in str(n)])  vs  sum(int(c) for c in str(n))
  앞: 리스트 만들고 → sum     (메모리에 리스트 생성)
  뒤: 바로 sum 에 전달          (메모리 효율 좋음)
  결과는 동일, 뒤가 더 Pythonic
```

## range() 숫자 반복

```python
for i in range(5):          # 0 1 2 3 4
for i in range(1, 6):       # 1 2 3 4 5
for i in range(0, 10, 2):   # 0 2 4 6 8  (2씩 증가)
for i in range(10, 0, -1):  # 10 9 8 ... 1  (역순)
```

## enumerate — 인덱스 + 값 동시에

```python
files = ["a.csv", "b.csv", "c.csv"]

for i, file in enumerate(files):
    print(i, file)
# 0 a.csv
# 1 b.csv
# 2 c.csv

# 1부터 시작
for i, file in enumerate(files, 1):
    print(i, file)
# 1 a.csv ...
```

---

---

# ③ while — 조건 반복

```python
while 조건:
    # 조건이 True 인 동안 계속 반복
```

```python
import time
retry = 0

while retry < 5:
    print(f"{retry + 1}번째 시도")
    retry += 1
    time.sleep(1)

print("종료")
```

```
⚠️ while True 에 break 없으면 무한 루프
   서버 비용 폭탄의 주범
   → 항상 탈출 조건 확인
```

---

---

# ④ break / continue

```
break    → 반복문 즉시 종료 (탈출)
continue → 이번 턴만 건너뛰고 다음으로
```

```python
for i in range(10):
    if i == 3:
        continue   # 3 건너뜀
    if i == 5:
        break      # 5 되면 종료
    print(i)
# 결과: 0 1 2 4
```

---

---

# ⑤ for-else — 파이썬 전용 문법

```
for 문이 break 없이 끝까지 돌면 else 실행
break 를 만나면 else 실행 안 됨

용도: "찾았는지 못 찾았는지" 를 Flag 변수 없이 처리
```

```python
# ❌ 기존 방식 — Flag 변수 필요
numbers = [1, 3, 5, 7]
is_found = False

for n in numbers:
    if n == 4:
        is_found = True
        break

if not is_found:
    print("4 없음")

# ✅ for-else — Flag 변수 불필요
for n in numbers:
    if n == 4:
        print("4 찾음!")
        break
else:
    print("4 없음")   # break 없이 끝까지 돌면 실행
```

---

---

# 자주 하는 실수

## ① 들여쓰기 에러

```python
# ❌ 들여쓰기 없으면 에러
if True:
print("hello")   # IndentationError

# ✅
if True:
    print("hello")
```

## ② for 문 안에서 리스트 수정

```python
my_list = [1, 2, 3, 4, 5]

# ❌ 순회하면서 원본 수정 → 인덱스 꼬임
for item in my_list:
    if item % 2 == 0:
        my_list.remove(item)   # 대참사

# ✅ 새 리스트에 담기
result = [item for item in my_list if item % 2 != 0]
```

## ③ list(str(n)) 불필요하게 씀

```python
n = 12345

# ❌ 불필요한 변환
for c in list(str(n)):
    ...

# ✅ str 은 바로 순회 가능
for c in str(n):
    ...

# ✅ 합계도 한 줄로
sum(int(c) for c in str(n))
```

---

---

# ⑥ := — 왈러스 연산자 (Python 3.8+)

## 한 줄 요약

```
할당 + 조건 검사를 동시에 하는 연산자
:= 모양이 바다코끼리 눈과 엄니 같아서 "왈러스(walrus)"

기존: 변수에 값 넣고 → 다음 줄에서 조건 검사
:=  : 조건 검사하면서 동시에 변수에 저장
```

## 기본 사용

```python
# 기존 방식
n = len(data)
if n > 10:
    print(n)

# := 방식 — 할당 + 조건 동시에
if (n := len(data)) > 10:
    print(n)   # n 을 여기서도 사용 가능
```

## while 루프 — 가장 많이 쓰는 패턴

```python
# 기존 방식
chunk = f.read(8192)
while chunk:
    process(chunk)
    chunk = f.read(8192)   # 루프 끝에 다시 읽어야 함 (중복)

# := 방식 — 중복 제거
while chunk := f.read(8192):
    process(chunk)
```

```python
# 입력 받으며 처리
while (line := input("입력: ")) != "quit":
    print(f"입력값: {line}")
# "quit" 입력 시 종료
```

## if + 재사용

```python
import re

# 기존 방식
m = re.search(r"\d+", text)
if m:
    print(m.group())

# := 방식
if m := re.search(r"\d+", text):
    print(m.group())   # m 바로 사용
```

## List Comprehension 에서

```python
data = [1, -2, 3, -4, 5]

# 기존: 같은 계산 두 번
[abs(x) for x in data if abs(x) > 2]

# := 방식: 계산 한 번만
[y for x in data if (y := abs(x)) > 2]
# [3, 4, 5]
```

## 주의사항

```python
# ⚠️ 괄호 필수 — 우선순위 때문
if n := len(data) > 10:    # ❌ n = (len(data) > 10) = True/False
    print(n)               # 원하는 결과 아님

if (n := len(data)) > 10:  # ✅ n = len(data) 먼저 할당
    print(n)               # 정상

# ⚠️ 남용 금지 — 읽기 어려워짐
# 간단한 경우는 기존 방식이 더 명확
n = len(data)
if n > 10:
    print(n)
```

```
언제 쓰면 좋은가:
  while 루프에서 반복 읽기 (파일 / 소켓 / 스트림)
  if 조건에서 결과를 바로 재사용할 때
  같은 함수 호출을 두 번 피하고 싶을 때

언제 쓰지 않는 게 나은가:
  단순 if 조건 (굳이 := 쓸 필요 없음)
  가독성보다 중요한 게 없는 경우
```