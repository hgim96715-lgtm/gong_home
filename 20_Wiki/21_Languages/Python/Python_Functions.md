---
aliases:
  - 함수
  - def
  - args
  - kwargs
  - 가변인자
  - 언패킹
  - 딕셔너리
  - 튜플
tags:
  - Python
related:
  - "[[Airflow_Operators]]"
  - "[[Airflow_Hooks]]"
  - "[[Python_Database_Connect]]"
  - "[[Python_Dictionaries]]"
---
## 개념 한 줄 요약

**"자주 쓰는 코드를 '부품'처럼 포장해서 이름을 붙인 것. (입력을 넣으면 → 뚝딱뚝딱 처리해서 → 출력을 뱉어냄)"**
**`*args`와 `**kwargs`** 는 입력값이 몇 개가 들어올지 모를 때 사용하는 **'마법의 주머니(가변 인자)'** 입니다.

---
## 왜 필요한가 (Why)

**문제점:**
1.  **반복:** 똑같은 로직(예: 날짜 포맷 변경)을 10번 써야 할 때, 복사-붙여넣기하면 수정할 때 10군데를 다 고쳐야 합니다.
2.  **Airflow의 상황:** Airflow는 태스크를 실행할 때 "실행 날짜(`ds`)", "태스크 인스턴스(`ti`)", "파라미터(`params`)" 등 수많은 정보를 함수에 던져주는데, **개발자가 이걸 일일이 다 파라미터로 적기엔 너무 많고 복잡합니다.**

**해결책:**
- **함수(`def`)** 로 로직을 묶어 관리합니다.
- **`**kwargs`** 를 사용해 Airflow가 던져주는 수십 개의 정보를 **딕셔너리 하나로 퉁쳐서** 받아냅니다.

---
## 함수 (Anatomy)

함수는 크게 **입력(Input), 처리(Process), 출력(Output)** 3단계로 이루어집니다.

```python
#      (함수 이름)   (입력 = 파라미터)
def make_coffee(bean_type, water):
    # [처리]
    print(f"{bean_type} 원두를 갑니다...")
    result = f"{bean_type} 커피 + 물 {water}ml"
    
    # [출력 = 반환값]
    return result
```

- **`def`**: "나 지금부터 함수 만든다!" 선언.
- **`return`**: "이게 내 결과물이야. 가져가!" (함수 종료).

---
## `return` vs `print` (가장 큰 착각)

초보자가 가장 많이 하는 실수입니다.

- **`print`**: 화면에 그냥 **보여주고 끝**입니다. (영수증 보여주기)
- **`return`**: 값을 **변수에 저장**할 수 있게 줍니다. (실제 물건 건네주기)

```python
def print_sum(a, b):
    print(a + b)  # 화면에 '5'가 보임

def return_sum(a, b):
    return a + b  # 값 5를 던져줌

# 테스트
x = print_sum(2, 3)
print(x)  # 결과: None (받은 게 없음!)

y = return_sum(2, 3)
print(y)  # 결과: 5 (값을 받아서 y에 저장함!)
```

---
## `*args` : "몇 개가 올지 몰라요" (튜플) 

입력값이 1개일 수도, 100개일 수도 있을 때 사용합니다. 
받은 값들을 **튜플(Tuple)** 로 묶어서 처리합니다.

- **상황:** "숫자를 다 더하는 함수를 만들어줘. 근데 숫자가 몇 개일지는 몰라."

```python
# *args: 입력값들을 'args'라는 튜플로 몽땅 받음
def add_all(*args):
    print(args)      # (1, 2, 3, 4, 5) <- 튜플로 들어옴!
    return sum(args) # 튜플 합계 계산

print(add_all(1, 2))          # 3
print(add_all(1, 2, 3, 4, 5)) # 15
```

---
## `**kwargs` : "이름표 붙여서 딕셔너리로 주세요"

입력값을 `키=값` 형태로 주면, 함수 내부에서는 **딕셔너리(Dictionary)** 로 받아서 씁니다. (Keyword Arguments의 약자)

- **상황:** "회원가입 함수를 만드는데, 이름/나이는 필수고 취미/주소/MBTI는 선택사항이야."

### ① 함수 정의할 때 (`**kwargs`로 받기)

```python
def introduce(**kwargs):
    print(kwargs)  # {'name': 'Gong', 'age': 20} <- 딕셔너리가 됨!
    
    if 'name' in kwargs:
        print(f"제 이름은 {kwargs['name']}입니다.")

# 사용법: 키=값 형태로 전달
introduce(name="Gong", age=20, city="Seoul")
```

### ② 함수 사용할 때 (`**`로 딕셔너리 풀기) ️

반대로, **이미 만들어진 딕셔너리**를 함수에 한 방에 넣고 싶을 때도 `**`를 씁니다.

```python
def save_user(name, age, city):
    print(f"{name}님({age}세)은 {city}에 삽니다.")

# 내 데이터가 딕셔너리로 있다면?
user_data = {'name': 'Kim', 'age': 30, 'city': 'Busan'}

# [방법 1] 하나씩 꺼내기 (귀찮음)
save_user(user_data['name'], user_data['age'], user_data['city'])

# [방법 2] 딕셔너리 언패킹 (초간단!) ⭐️
# "user_data 딕셔너리를 풀어서(Unpack) 인자에 쇽쇽 넣어줘!"
save_user(**user_data)
```

**데이터 엔지니어링 팁:** DB 설정(`DB_CONFIG`)이나 API 요청 보낼 때 이 **언패킹(`**`)** 기술을 정말 많이 씁니다.

---
## 기본값 설정 (Default Parameter)

값을 안 넣었을 때 사용할 **기본값**을 정해둘 수 있습니다.

```python
# tax_rate를 안 넣으면 자동으로 0.1로 계산
def calc_price(price, tax_rate=0.1):
    return price * (1 + tax_rate)

print(calc_price(10000))       # 11000.0 (기본값 0.1 적용)
print(calc_price(10000, 0.2))  # 12000.0 (0.2 적용)
```

**주의:** 기본값이 있는 파라미터(`tax_rate`)는 무조건 **맨 뒤**에 와야 합니다.

---
## Type Hinting

코드를 읽기 좋게 "이건 숫자야", "이건 문자야"라고 힌트를 적어주는 것이 좋습니다. (강제성은 없음)

```python
# name은 str이고, age는 int이고, 결과는 str일 것이다.
def greeting(name: str, age: int) -> str:
    return f"{name} is {age} years old."
```

---
## 요약 (Cheat Sheet)

|**문법**|**의미**|**자료형(내부)**|**예시**|
|---|---|---|---|
|`def func(a, b)`|일반 파라미터|변수|`func(1, 2)`|
|`def func(a=10)`|기본값 파라미터|변수|`func()` 또는 `func(5)`|
|`def func(*args)`|개수 무제한 (값만)|**튜플 (Tuple)**|`func(1, 2, 3)`|
|`def func(**kwargs)`|개수 무제한 (키=값)|**딕셔너리 (Dict)**|`func(a=1, b=2)`|
|`func(**my_dict)`|딕셔너리 언패킹|-|`func(**{'a':1})`|


---
## 초보자가 자주 착각하는 포인트

1. **순서가 생명이다:**
    - 파이썬에서 인자를 섞어 쓸 때는 반드시 **`일반 변수` -> `*args` -> `**kwargs`** 순서를 지켜야 합니다.
    - `def func(**kwargs, a)` -> **에러 발생!** 🚨 무조건 `**kwargs`가 맨 마지막입니다.

2.  **별표(`*`)가 핵심이다:**
    - 변수명을 꼭 `args`, `kwargs`로 안 써도 됩니다. `*data`, `**context`라고 써도 작동합니다.
    - 하지만 전 세계 개발자들의 약속(Convention)이므로 웬만하면 `args`, `kwargs`를 쓰세요.
        
3.  **`print`와 `return`의 차이:**
    - `print`는 화면에 보여줄 뿐이고, 함수 밖으로 값을 전달하지 못합니다.
    - 다른 변수에 결과를 저장하고 싶다면 반드시 `return`을 써야 합니다.
