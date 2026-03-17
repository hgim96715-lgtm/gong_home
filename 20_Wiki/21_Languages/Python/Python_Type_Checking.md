---
aliases:
  - Python_Type_Checking
  - isinstance
  - type
  - 자료형_검사
tags:
  - Python
related:
  - "[[Python_Classes_Objects]]"
  - "[[00_Python_HomePage]]"
  - "[[Python_Variables_Types]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Dictionaries]]"
---
# Python_Type_Checking — type vs isinstance

## 한 줄 요약

```
"이 변수, 내가 생각하는 그 데이터 타입 맞아?"
데이터의 자료형을 확인하는 내장 함수
```

---

---

# 왜 필요한가?

```
외부 API / 파일에서 읽어온 데이터는 타입을 확신할 수 없음
→ 에러 터지기 전에 안전장치(방어적 프로그래밍) 필요

"리스트가 맞을 때만 반복문 돌아라"
"dict 가 맞을 때만 .get() 써라"
```

---

---

# ① isinstance — 타입 검사 (권장)

```python
isinstance(검사할_값, 확인하고_싶은_타입)
# 맞으면 True / 틀리면 False
```

```python
data = [1, 2, 3]

isinstance(data, list)   # True
isinstance(data, dict)   # False
```

## 여러 타입 동시 검사 — 튜플로 묶기

```python
val = 3.14

# or 여러 번 쓸 필요 없이 튜플로 한 번에
isinstance(val, (int, float))    # True  ← int 또는 float
isinstance(val, (str, bool))     # False
```

```python
# 실전: API 응답 데이터 방어적 처리
response = fetch_data()

if isinstance(response, list):
    for item in response:       # 리스트일 때만 반복
        process(item)
elif isinstance(response, dict):
    process(response)           # 딕셔너리면 바로 처리
else:
    print(f"예상치 못한 타입: {type(response)}")
```

---

---

# ② type — 타입 확인

```python
type(3)          # <class 'int'>
type(3.14)       # <class 'float'>
type("hello")    # <class 'str'>
type([1, 2])     # <class 'list'>

# 타입 비교
type(3) == int   # True
```

---

---

# ③ type vs isinstance — 핵심 차이

```
type()       깐깐함 → "정확히 그 클래스" 여야만 True
isinstance() 유연함 → "그 클래스의 자식(상속)" 도 True
```

```python
class Animal:
    pass

class Dog(Animal):   # Animal 을 상속받은 Dog
    pass

baduk = Dog()

# type() — 상속 무시
type(baduk) == Dog      # True
type(baduk) == Animal   # False  ← 개는 동물 아니라고?? 🚨

# isinstance() — 상속 포함
isinstance(baduk, Dog)     # True
isinstance(baduk, Animal)  # True  ← 개도 동물의 한 종류 ✅
```

```
결론:
  파이썬 공식 문서에서도 isinstance() 권장
  상속 관계 고려 → 더 안전하고 유연함

  type() 쓸 때:
  "정확히 이 클래스 그 자체인지" 확인할 때만 사용
```

---

---

# 한눈에 비교

|항목|`type()`|`isinstance()`|
|---|---|---|
|상속 인정|❌|✅|
|여러 타입 동시 검사|❌|✅ `(int, float)`|
|공식 권장|-|✅|
|용도|정확한 클래스 확인|타입 안전성 검사|