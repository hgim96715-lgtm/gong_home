---
aliases:
  - 데코레이터
  - Decorator
  - "@"
  - TakFlow
tags:
  - Python
  - Airflow
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Python_Functions]]"
  - "[[Airflow_XComs]]"
---
## 개념 한 줄 요약

**데코레이터(Decorator)** 는 선물 상자의 **"포장지"** 같은 거야.
내용물(기존 함수)은 건드리지 않고, 겉에다가 예쁜 포장이나 리본(추가 기능)을 덧붙여서 **새로운 기능**을 입히는 문법이지. 
기호는 골뱅이(**`@`**)를 써.

---
## 왜 필요한가 (Why)

**문제점 (반복 노동):**
- 함수가 100개 있는데, 모든 함수가 실행될 때마다 "실행 시작 시간"과 "끝난 시간"을 로그로 남기고 싶어. 
- 100개 함수에 `print` 문을 일일이 다 복사해 넣을 거야? (유지보수 지옥)

**해결책:**
- "시간 측정 기능"을 가진 데코레이터를 하나 만들어두고, 필요한 함수 위에 스티커 붙이듯 `@timer`만 붙이면 끝!

---
## Practical Context (Airflow 2.0의 혁명) 

우리가 이 어려운 걸 배우는 이유는 딱 하나, **Airflow TaskFlow API** 때문이야.

**옛날 방식 (Classic):**
- PythonOperator`를 import하고, `python_callable`에 함수 이름을 넣고... 코드가 길고 복잡했어.

**요즘 방식 (Decorator):**
- 그냥 파이썬 함수 위에 **`@task`** 라고 붙이면, Airflow가 알아서 "아, 이건 그냥 함수가 아니라 **Airflow Operator**구나!" 하고 변신시켜 줘.

```python
# 일반 파이썬 함수가 Airflow Task로 변신하는 마법!
@task
def extract_data():
    return "data"
```

---
## Code Core Points (How it works)

데코레이터의 내부 구조는 **"함수를 받아서, 함수를 리턴하는 함수"** 야. (말이 어렵지? 코드로 보자.)

### A. 기본 구조 (샌드위치 만들기)

```python
# 1. 데코레이터 정의 (포장지 공장)
def my_decorator(func):
    def wrapper():
        print("🎁 포장지를 쌉니다 (함수 실행 전)")
        func() # 원래 함수 실행
        print("🎀 리본을 묶습니다 (함수 실행 후)")
    return wrapper

# 2. 적용하기 (골뱅이 붙이기)
@my_decorator
def say_hello():
    print("Hello!")

# 3. 실행
say_hello()
```

**실행 결과:**

```text
🎁 포장지를 쌉니다 (함수 실행 전)
Hello!
🎀 리본을 묶습니다 (함수 실행 후)
```

---
## Detailed Analysis (Airflow에서의 활용)

Airflow에서 자주 보게 될 데코레이터 3대장이야.

****`@dag`**:
- "이 함수는 단순 함수가 아니라, **DAG 파이프라인 그 자체**야." 라고 선언.

**`@task`**:
- "이 함수는 **PythonOperator**로 변환되어야 해." 
- (XCom 데이터 전달도 자동으로 해줌!)

**`@task_group`**:
- "이 함수 안에 있는 태스크들은 **폴더(그룹)** 로 묶어서 보여줘." ([[Airflow_TaskGroup]] 참고)

---
##  Advanced Tip: 함수 정의 위치 (Inside vs Outside)

"Airflow에서 자주 보게 될 데코레이터 3대장 함수를 꼭 `with DAG` 안에서 만들어야 하나요?" 
👉 **아니! 밖에 만들어도 돼.**

### A. 차이점 이해하기

* **Outside (DAG 밖):** "도장(Stamp)을 파는 행위"
    * 함수를 미리 만들어두는 거야. 아직 실행 계획은 없어.
    * **장점:** 다른 DAG 파일에서 `from my_tasks import generate_number` 처럼 **가져다 쓸 수 있음(재사용성).**

* **Inside (DAG 안):** "도장을 종이에 찍는 행위"
    * 만들어진 함수를 DAG 흐름 안에 배치하는 거야.

### B. 코드 예시

```python
from airflow.decorators import task

# 1. 밖에서 정의 (Definition) - 깔끔하게 기능만 구현
# 이 함수는 dag1에서도 쓰고, dag2에서도 쓸 수 있어!
@task
def generate_number():
    return 100
    
@task 
def read_number(num): 
	print(num)

with DAG(dag_id='my_dag', ...) as dag:
    # 2. 안에서 호출 (Invocation) - 조립만 수행
    # 이때 비로소 "이 DAG의 태스크"가 됨
    generate_number()
    
    # 이렇게 밖에서 만든 걸 안에서 합칠 때 "체이닝"을 쓰면 예술이야!
    read_number(generate_number())
```

>파이썬 입장에서 생각해보면 쉬워:
>**`def` (함수 정의)** 는 그냥 "이런 기능이 있어"라고 컴퓨터한테 알려주는 거고,
>**`func()` (함수 호출)** 이 실제로 "지금 실행해!"(Airflow에선 "DAG에 등록해!")라고 하는 거니까.
>**공통으로 쓰는 함수(예: 슬랙 알림 보내기, 전처리 모듈)** 는 밖에다 정의하거나 아예 다른 파일(`utils.py`)에 두고 `import` 해서 쓰는 게 **고수들의 방식**이야!

>**연결된 지식:** 밖에서 만든 함수를 안에서 **한 줄로 우아하게 연결하는 방법**이 궁금하다면?
> 👉 **[[Airflow_XComs#2. 심화: 한 줄로 끝내기 (Task Chaining) ✨|심화: 한 줄로 끝내기 (Task Chaining) 예제 보러가기]]**
---
## 초보자가 자주 착각하는 포인트

1. "함수 이름이 바뀌나요?"
	- 엄밀히 말하면 바뀌어! `say_hello`를 호출했지만 실제로는 `wrapper`가 실행된 거잖아.
	- 그래서 디버깅할 때 헷갈릴 수 있어. (이를 방지하기 위해 `functools.wraps`라는 걸 쓰지만, 지금은 몰라도 됨)

2. "여러 개 붙여도 되나요?"
	- 응! `@task` 위에 `@retry` 붙이고 그 위에 `@timer` 붙이고... 층층이 쌓을 수 있어. 
	- (위에서부터 차례대로 포장됨) 
	- **코드 읽는 순서** = 위 → 아래
	- **실제 실행 순서 / 함수 호출 순서** = 아래 → 위

3. "이거 몰라도 Airflow 할 수 있나요?"
	- 아니. Airflow 2.0 이상을 쓴다면 **선택이 아니라 필수**야. 코드가 획기적으로 줄어들거든.


