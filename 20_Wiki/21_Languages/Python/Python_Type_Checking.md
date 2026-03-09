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
# Type vs Instance

## 개념 한 줄 요약

**"이 변수, 내가 생각하는 그 데이터 타입 맞아?"**
데이터의 자료형을 확인할 때 쓰는 파이썬 내장 함수입니다.

---
## 왜 필요한가? (Why)

데이터 엔지니어링 실무에서는 외부 API나 파일에서 데이터를 읽어오기 때문에, 이 데이터가 딕셔너리(`dict`)인지 리스트(`list`)인지 확신할 수 없는 경우가 많습니다. 이때 에러가 터지기 전에 **"리스트가 맞을 때만 반복문을 돌아라"** 처럼 안전장치(방어적 프로그래밍)를 걸기 위해 반드시 필요합니다.

---
## Code Core Points

### 기본 문법 (`isinstance`)

- `isinstance(검사할_값, 확인하고_싶은_타입)`
- 맞으면 `True`, 틀리면 `False`를 반환합니다.

```python
data = [1, 2, 3]

print(isinstance(data, list))  # True
print(isinstance(data, dict))  # False
```

### 여러 타입 동시에 검사하기 (Tuple 활용)

"문자열이거나 정수면 통과시켜!" 라고 할 때, `or` 연산자를 여러 번 쓸 필요 없이 튜플 `()`로 묶어서 한 방에 검사할 수 있습니다.

```python
val = 3.14

# "int 랑 float 둘 중 하나라도 맞으면 True!"
if isinstance(val, (int, float)):
    print("숫자형 데이터입니다. 계산을 시작합니다.")
```

---
## Detailed Analysis: `type()` vs `isinstance()`

타입을 검사할 때 초보자는 `type(a) == list`를 쓰고, 고수는 `isinstance(a, list)`를 씁니다. 
가장 결정적인 차이는 **"상속(Inheritance)을 인정하느냐"** 입니다.

- **`type()`:** 아주 깐깐합니다. **"정확히 그 클래스"** 여야만 `True`를 줍니다.
- **`isinstance()`:** 유연합니다. **"그 클래스의 자식(상속받은 녀석)이어도"** `True`를 줍니다. (다형성 존중)

```python
class Animal:
    pass

class Dog(Animal): # Animal을 상속받은 Dog
    pass

baduk = Dog()

# 1. type()의 깐깐함
print(type(baduk) == Dog)    # True (너 개 맞네)
print(type(baduk) == Animal) # False (너 동물 아니잖아? 개 잖아!) -> 🚨 논리적 오류 발생

# 2. isinstance()의 유연함
print(isinstance(baduk, Dog))    # True
print(isinstance(baduk, Animal)) # True (개도 동물의 한 종류지!) -> 👍 실무 권장
```

>**결론:** 파이썬 공식 문서에서도 데이터 타입을 검사할 때는 `type()` 비교보다 **`isinstance()` 사용을 강력하게 권장**합니다!

