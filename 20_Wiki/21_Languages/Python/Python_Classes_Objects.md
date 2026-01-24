---
aliases:
  - Class
  - Object
  - Instance
  - 클래스
  - 객체
  - 인스턴스
  - 붕어빵
tags:
  - Python
  - OOP
related:
  - "[[Python_Inheritance]]"
  - "[[Python_Dictionaries]]"
---
## 개념 한 줄 요약

**클래스(Class)** 는 제품을 찍어내는 **"설계도(붕어빵 틀)"** 이고,
**객체(Object/Instance)** 는 그 설계도로 만들어진 **"실체(슈크림 붕어빵, 팥 붕어빵)"** 야.

---
## 왜 필요한가 (Why)

**문제점:**
- 똑같은 기능을 하는 로봇을 100개 만들어야 하는데, 그때마다 코드를 복사-붙여넣기 할 거야? (유지보수 지옥 )

**해결책:**
- "로봇 설계도(Class)"를 하나 잘 만들어두고, 필요할 때마다 `robot1`, `robot2` 하고 찍어내기만 하면 돼.

---
## Core Points (붕어빵 이론) 

### A. 문법 구조

* **`class`**: 설계도 이름 (보통 첫 글자 대문자)
* **`__init__`**: 붕어빵이 처음 태어날 때(생성될 때) 속 재료를 넣는 곳.
* **`self`**: "나 자신"을 가리키는 말. (이 붕어빵의 팥은 내 거야!)

```python
# 1. 설계도 만들기 (Class)
class Bungeoppang:
    # 생성자 (초기화)
    def __init__(self, taste, price):
        self.taste = taste  # 속 재료 (팥, 슈크림...)
        self.price = price  # 가격

    # 행동 (메서드)
    def eat(self):
        print(f"냠냠! {self.taste} 맛이 나요.")

# 2. 실제 붕어빵 찍어내기 (Instance)
fish1 = Bungeoppang("팥", 1000)      # 팥 붕어빵 탄생!
fish2 = Bungeoppang("슈크림", 1500)  # 슈크림 붕어빵 탄생!
```

---
## Deep Dive: Dictionary vs Class (네가 헷갈렸던 것!)

이게 제일 중요해! 
데이터를 담는다는 점은 비슷하지만, **꺼내는 도구**가 달라.

|**구분**|**Dictionary (딕셔너리)**|**Class Object (객체)**|
|---|---|---|
|**비유**|서랍장 (이름표 붙음)|로봇 (팔다리 달림)|
|**생성**|`{ "taste": "팥" }`|`Bungeoppang("팥")`|
|**접근법**|**대괄호 `['key']`**|**점 `.attribute`**|
|**특징**|데이터만 담음|데이터 + **행동(함수)**도 있음|

```python
# [상황] 팥 정보를 꺼내고 싶다!

# 1. 딕셔너리일 때
my_dict = {"taste": "팥"}
print(my_dict["taste"])  # ⭕️ 정답 (서랍 열기)
# print(my_dict.taste)   # ❌ 에러! (서랍장에 팔다리가 어딨어?)

# 2. 객체일 때
my_obj = Bungeoppang("팥", 1000)
print(my_obj.taste)      # ⭕️ 정답 (로봇 팔 조작)
my_obj.eat()             # ⭕️ 정답 (로봇아 먹어라!)
# print(my_obj["taste"]) # ❌ 에러! (로봇은 서랍이 아냐)
```

---
## Practical Context (Airflow Operator)

우리가 매일 쓰는 `BashOperator`도 사실은 다 **클래스**야!

```python
from airflow.operators.bash import BashOperator

# BashOperator라는 '설계도'를 가지고
# t1이라는 '실체(인스턴스)'를 생성하는 과정
t1 = BashOperator(
    task_id='print_date',
    bash_command='date'
)

# 그래서 t1.task_id 처럼 점(.)으로 속성을 꺼낼 수 있는 거야.
print(t1.task_id)
```

---
## 초보자가 자주 착각하는 포인트

1. **"self는 왜 자꾸 들어가요?"**
	- 함수 안에 `self`가 없으면, 이게 "누구의" 팥인지 몰라.
	- `fish1.taste`인지 `fish2.taste`인지 구별하기 위해 **"지금 부르는 놈 자신(self)"** 이라고 명찰을 달아주는 거야.

2. "`__init__`이 뭐예요?"
	- **Initialize(초기화)** 의 약자야. 
	- 붕어빵 틀에서 빵이 `짠!` 하고 나올 때 무조건 실행되는 **"탄생 의식"** 함수라고 생각하면 돼.

>**중괄호 `{}`** 로 만들었으면 ➡ **`[]`** 로 꺼낸다.
>**대문자 `Class()`** 로 만들었으면 ➡ **`.`** 으로 꺼낸다.