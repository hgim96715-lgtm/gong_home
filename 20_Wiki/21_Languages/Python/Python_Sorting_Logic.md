---
aliases:
  - Sorted
  - Sort
  - Lambda
  - 람다
  - 딕셔너리정렬
  - key옵션
  - 튜플정렬
  - key
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_String_Methods]]"
  - "[[Python_Dictionaries]]"
  - "[[Python_Lambda_Map]]"
---

# Python_Sorting_Logic — 정렬

## 한 줄 요약

```
순서가 뒤죽박죽인 데이터를
내가 원하는 '기준(key)' 을 정해서 줄 세우는 기술
```

---

---

# ① sorted() vs sort()

```python
sorted(데이터, key=기준함수, reverse=True/False)
리스트.sort(key=기준함수, reverse=True/False)
```

|특징|`sorted()`|`.sort()`|
|---|---|---|
|원본 변경|❌ 안 바뀜 (안전)|✅ 바뀌어버림 (주의)|
|반환값|정렬된 새 리스트|None|
|사용 대상|모든 iterable (리스트·튜플·딕셔너리 등)|리스트만|
|체이닝|가능|불가 (None 반환이라)|

```python
numbers = [3, 1, 2]

# ❌ 흔한 실수 — sort() 결과를 변수에 담으면 None
result = numbers.sort()
print(result)   # None 😱

# ✅ sort() 는 그냥 혼자 실행
numbers.sort()
print(numbers)  # [1, 2, 3]

# ✅ 원본 유지하려면 sorted()
safe = sorted([3, 1, 2])
print(safe)     # [1, 2, 3]
```

---

---

# ② key 옵션 — 정렬 기준 지정

```
컴퓨터는 ('A', 30) 덩어리를 주면
무조건 맨 앞 'A' 만 보고 정렬함

key 는 "앞에 있는 거 보지 말고, 뒤 숫자(30) 기준으로 봐!" 라고 지정하는 것
```

```python
data = {'A': 30, 'C': 5, 'B': 15}

# key 없으면 → 튜플 첫 번째(알파벳) 기준 정렬
sorted(data.items())
# [('A', 30), ('B', 15), ('C', 5)]  ← 알파벳 순

# key 있으면 → 값(숫자) 기준 정렬
sorted(data.items(), key=lambda item: item[1], reverse=True)
# [('A', 30), ('B', 15), ('C', 5)]  ← 숫자 큰 순
```

```
key=lambda item: item[1]
  → "원소(item) 하나가 들어오면 [1]번째를 기준으로 삼아라"
```

>lambda 는 [[Python_Lambda_Map]] 참고

---

---

# ③ reverse 옵션

```
기본값(False) = 오름차순  1, 2, 3... / 가, 나, 다...
reverse=True  = 내림차순  3, 2, 1...
```

```python
scores = [50, 90, 70]

sorted(scores)               # [50, 70, 90]  오름차순
sorted(scores, reverse=True) # [90, 70, 50]  내림차순 (랭킹, TOP N)
```

---

---

# ④ 튜플 정렬 — 사전식(Lexicographic) 정렬

```
파이썬은 튜플을 만나면
맨 앞(0번째) 값부터 차례대로 비교하는 '본능' 이 있음

1. 첫 번째 값으로 줄 세우기
2. 첫 번째 값이 같으면(동점) → 두 번째 값으로 마저 줄 세우기
3. 두 번째도 같으면 → 세 번째 값으로...
```

```python
data = [(1, 'C'), (3, 'A'), (1, 'B')]

sorted(data)
# [(1, 'B'), (1, 'C'), (3, 'A')]
#   ↑ 1이 두 개라 동점 → 두 번째 값 B, C 로 결정
```

### 튜플 본능 활용 — key=lambda 없이 정렬

```
튜플 만들 때 정렬하고 싶은 기준을 맨 앞에 두면
lambda 없이도 알아서 정렬됨

(가장 중요한 정렬 기준, 두 번째 기준, 그냥 들고 갈 데이터)
```

```python
# 예: 출석한 학생만 랭크 순서로 뽑기
rank       = [3, 1, 2]
attendance = [True, False, True]

# 애초에 (랭크, 인덱스) 순서로 튜플 만들기
arr = [(x, i) for i, x in enumerate(rank) if attendance[i]]
# [(3, 0), (2, 2)]

arr = sorted(arr)
# [(2, 2), (3, 0)]  ← 랭크 기준 자동 정렬, lambda 불필요
```

---

---

# ⑤ 딕셔너리 정렬 — 실전 패턴

```python
data = {'A': 30, 'C': 5, 'B': 15}

# 값(value) 기준 내림차순
sorted(data.items(), key=lambda x: x[1], reverse=True)
# [('A', 30), ('B', 15), ('C', 5)]

# 키(key) 기준 오름차순
sorted(data.items(), key=lambda x: x[0])
# [('A', 30), ('B', 15), ('C', 5)]

# 값 기준 정렬 후 딕셔너리로 복원
dict(sorted(data.items(), key=lambda x: x[1], reverse=True))
# {'A': 30, 'B': 15, 'C': 5}
```

---

---

# 핵심 정리

```
sorted()   원본 유지, 새 리스트 반환  → 거의 항상 이걸 쓸 것
.sort()    원본 변경, None 반환       → 변수에 담으면 None 함정

key        정렬 기준 지정
           단순 숫자/문자  → key 생략
           튜플·딕셔너리   → key=lambda x: x[1]
           튜플 맨 앞이 기준이면 → lambda 없이 sorted() 그대로

reverse=True   내림차순 (랭킹, TOP N 뽑을 때)

데이터 엔지니어 반사 공식:
  "빈도수 많은 순서대로" → sorted(..., key=lambda x: x[1], reverse=True)
```