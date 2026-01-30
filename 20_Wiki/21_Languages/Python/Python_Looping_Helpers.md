---
aliases:
  - range
  - enumerate
  - zip
  - 반복문도구
  - 파이썬반복
tags:
  - Python
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Control_Flow]]"
  - "[[PySpark_Session_Context]]"
  - "[[00_Python_HomePage]]"
---
## 개념 한 줄 요약

**"반복문(`for`)을 더 똑똑하고 편하게 만들어주는 3가지 필수 아이템."**

1.  **`range()`**: 숫자 범위를 만들어줌. (0, 1, 2...)
2.  **`enumerate()`**: 번호표(Index)를 같이 붙여줌.
3.  **`zip()`**: 두 개의 리스트를 지퍼처럼 잠가서 합쳐줌.

---
## 숫자 제조기: `range()` 🔢

데이터 엔지니어링에서 **"테스트 데이터 100만 개 만들어봐"** 할 때 가장 먼저 쓰는 함수입니다.

### ① 기본 문법

`{python}range(시작, 끝, 보폭), 끝 숫자는 포함 안함`

```python
# range(시작, 끝, 보폭)
# * 끝 숫자는 포함하지 않음! (Stop - 1)

# 0부터 4까지 (총 5개)
r1 = range(5) 

# 2부터 10까지, 2칸씩 점프 (2, 4, 6, 8, 10)
r2 = range(2, 11, 2)
```

### ② 얘도 "게으른 녀석"이다? 

 `map`처럼, `range`도 숫자를 미리 다 만들어두지 않습니다. 
 **"달라고 할 때 하나씩 줍니다."** (메모리 절약)

```python
r = range(999999999999) 
# 이렇게 큰 숫자를 써도 컴퓨터가 안 멈춤! (메모리 안 먹음)

print(r)       # 출력: range(0, 999999999999) (안 보여줌)
print(list(r)) # 🚨 주의! 이러면 컴퓨터 멈춤 (메모리 폭발)
```

---
## 번호표 붙이기: `enumerate()` 

로그 남길 때 필수입니다. **"몇 번째 데이터 처리 중인지"** 알아야 하니까요.

```python
names = ["Spark", "Airflow", "Docker"]

# (X) 촌스러운 방식
i = 0
for name in names:
    print(i, name)
    i += 1

# (O) 세련된 방식 (Index와 Value를 동시에!)
for i, name in enumerate(names):
    print(f"{i}번째 기술: {name}")
    
# 출력:
# 0번째 기술: Spark
# 1번째 기술: Airflow
# 2번째 기술: Docker
```

---
## 합체하기: `zip()` 

두 개의 리스트를 **같은 순서끼리** 묶어줍니다.

```python
tools = ["Spark", "Airflow"]
langs = ["Scala", "Python"]

# 지퍼(zip) 잠그기
pairs = zip(tools, langs)

# 딕셔너리로 만들기 딱 좋음!
print(dict(pairs))
# 결과: {'Spark': 'Scala', 'Airflow': 'Python'}
```

---
## Spark에서의 활용 (RDD 실습 복습)

```python
sc.parallelize( range(1000) )
```

- **`range(1000)`**: 0~999까지 숫자를 생성할 **준비**만 함. (메모리 거의 0)
- **`sc.parallelize(...)`**: 그 준비된 `range` 객체를 받아서, 여러 노드(컴퓨터)로 쫙 뿌리면서(Distributed) 실제 데이터를 만듦.



###  팁

"파이썬 초보 티를 벗는 가장 빠른 방법은 `range(len(list))` 대신 **`enumerate()`** 를 쓰는 거야. 
그리고 **`range`** 는 리스트가 아니라는 거! 
`print(range(5))` 했을 때 `[0, 1, 2, 3, 4]`가 안 나온다고 당황하지 말고, **'아, 이 녀석도 게으른 녀석이구나, `list()`를 씌워야겠네'** 라고 생각하면 100점이야!"

