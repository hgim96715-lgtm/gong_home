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
  - "[[Spark_Core_Broadcast]]"
---
## 개념 한 줄 요약 

**"관련 있는 데이터(변수)와 행동(함수)을 한 박스에 묶어놓은 것."**

* **클래스(Class):** 제품 설계도 (붕어빵 **틀**)
* **객체(Object/Instance):** 설계도로 찍어낸 실체 (팥 **붕어빵**, 슈크림 **붕어빵**)
* **속성(Attribute):** 객체 안에 들어있는 **데이터** (팥, 밀가루, 가격) 👈 **`.value`의 정체!**
* **메서드(Method):** 객체가 할 수 있는 **행동** (먹기, 굽기)

---
## 왜 필요한가? (Why) 

**[문제 상황]**
똑같은 기능을 하는 로봇 100개를 만들어야 합니다.
딕셔너리로 만들면, 로봇이 움직이는 함수(`move_robot`)를 따로 만들어서 매번 연결해줘야 합니다. (관리 지옥 🔥)

**[해결책]**
**"로봇 설계도(Class)"** 하나만 완벽하게 짜두면, `robot1 = Robot()`, `robot2 = Robot()` 처럼 한 줄로 복제할 수 있습니다.
게다가 데이터(`name`)와 행동(`move`)이 한 뭉치로 다닙니다.

---
## 핵심 해부 (Anatomy) 

이 구조를 모르면 **"점(.)"** [초중요] 점(`.`) vs 대괄호(`[]`)을 이해할 수 없습니다.

```python
# 1. 설계도(Class) 정의
class Bungeoppang:
    
    # ① 생성자 (__init__): 태어날 때 무조건 실행되는 '초기 세팅'
    def __init__(self, contents, price):
        self.contents = contents  # 속성 1 (데이터)
        self.price = price        # 속성 2 (데이터)

    # ② 메서드 (Method): 이 객체의 행동
    def eat(self):
        print(f"{self.contents} 맛 붕어빵을 먹어요! 냠냠.")

# -------------------------------------------------

# 2. 객체(Instance) 생성 -> "실체화"
fish1 = Bungeoppang("팥", 1000)      # 팥 붕어빵 탄생
fish2 = Bungeoppang("슈크림", 1500)  # 슈크림 붕어빵 탄생
```

---
## [내가 헷갈려하는것 ] 점(`.`) vs 대괄호(`[]`)

스파크에서 `broadcast.value` 때문에 헷갈리셨죠? 여기서 완벽하게 정리합니다.

|**구분**|**딕셔너리 (Dictionary)**|**객체 (Object)**|
|---|---|---|
|**생김새**|`my_dict = {"name": "A"}`|`my_obj = ClassName()`|
|**비유**|이름표 붙은 **서랍장**|팔다리 달린 **로봇**|
|**데이터 꺼낼 때**|**대괄호 `['key']`**|**점 `.attribute`**|
|**행동(함수)**|없음 (데이터만 저장)|있음 (`.method()`)|
|**예시**|`data['price']`|`fish1.price`|

**절대 규칙 (Golden Rule)**

- **중괄호 `{}`** 로 만들었으면 ➡ **`['키']`** 로 꺼낸다.
- **대문자 `Class()`** 로 만들었으면 ➡ **`.속성`** 으로 꺼낸다.

---
## 초보자 3대 난관 (FAQ)

### ① `__init__`이 뭐예요?

- **"Initialize(초기화)"** 의 약자입니다.
- 붕어빵이 틀에서 `짠!` 하고 나오는 순간, **자동으로 실행**되는 함수입니다.
- 여기서 팥을 넣을지(`self.contents`), 가격표를 붙일지(`self.price`) 정합니다.

### ② `self`는 도대체 왜 자꾸 써요? 🤬

- **"자기 자신"** 을 가리키는 손가락입니다.
- `fish1`과 `fish2`는 같은 설계도에서 나왔지만 서로 다른 존재죠?
- 컴퓨터에게 **"지금 말하는 `contents`는 `fish1` 꺼야!"** 라고 알려주기 위해 `self`를 붙입니다.
    - `fish1.eat()`을 실행하면 ➡ `self`는 `fish1`이 됩니다.
    - `fish2.eat()`을 실행하면 ➡ `self`는 `fish2`가 됩니다.

### ③ `.value` 같은 건 어디서 온 거예요?

- **클래스 안에서 `self.value = ...` 라고 정의해뒀기 때문**입니다.
- 스파크의 `Broadcast` 클래스를 뜯어보면 내부에 `self.value` 라는 변수가 숨어있습니다. 우리는 점(`.`)을 찍어서 그 변수를 꺼내 쓰는 겁니다.

---
## 실전 예제 (Data Engineering Context) 

우리가 맨날 쓰는 **Airflow Operator**도 사실 다 클래스입니다.

```python
from airflow.operators.bash import BashOperator

# 1. BashOperator 클래스(설계도)를 이용해
# 2. t1 이라는 객체(인스턴스)를 생성함
t1 = BashOperator(
    task_id='print_date',
    bash_command='date'
)

# 3. t1 안에 있는 속성(Attribute) 꺼내기 -> 점(.) 사용!
print(t1.task_id)      # 결과: 'print_date'
print(t1.bash_command) # 결과: 'date'
```



