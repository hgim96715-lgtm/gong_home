---
aliases:
  - 함수
  - def
  - args
  - kwargs
  - 가변인자
tags:
  - Python
related:
  - "[[Airflow_DAG_Operators]]"
  - "[[Airflow_Hooks]]"
---
## 개념 한 줄 요약

**함수(Function)** 는 특정 작업을 수행하는 코드 덩어리에 이름을 붙여 재사용하는 것이며, 
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
## Practical Context (Airflow 필수!)

데이터 엔지니어가 이 문법을 모르면 **`PythonOperator`** 를 제대로 쓸 수 없습니다.

```python
# Airflow에서 가장 흔한 패턴
def my_task_logic(**kwargs):
    # Airflow가 던져준 모든 정보가 kwargs라는 딕셔너리에 들어있음
    execution_date = kwargs['ds'] 
    ti = kwargs['ti']
    print(f"이 태스크는 {execution_date} 데이터를 처리합니다.")

task = PythonOperator(
    task_id='example_task',
    python_callable=my_task_logic,
    provide_context=True # (구버전) 모든 정보를 kwargs로 넘겨라!->지금은 이거 필요 없음
)
```

---
## 코드 심화 포인트

- **`def`**: 함수 정의 시작 키워드.
- **`return`**: 함수의 결과를 뱉어내는 곳. (이게 없으면 `None`을 반환)
- **`*args` (Arguments)**: 입력값들을 **튜플(Tuple)** 로 묶어서 받음. (순서 중요)
- **`**kwargs` (Keyword Arguments)**: 입력값들을 **딕셔너리(Dictionary)** 로 묶어서 받음. (이름 중요)

---
## 상세 분석

### 1. 기본 함수와 `return`

```python
def add(a, b):
    result = a + b
    return result

# 사용
val = add(3, 5) # val에는 8이 저장됨
```

### 2. `*args` : 갯수 무제한 (튜플)

인자가 1개일지 100개일지 모를 때 씁니다.

```python
def sum_all(*args):
    print(args)  # (1, 2, 3) <-- 튜플로 들어옴
    return sum(args)

print(sum_all(1, 2, 3))    # 6
print(sum_all(1, 2, 3, 4)) # 10
```

### 3. `**kwargs` : 이름표가 있는 무제한 (딕셔너리) 

**Airflow가 사랑하는 문법**입니다. 
"키=값" 형태로 들어오는 모든 것을 딕셔너리로 만듭니다.

```python
def print_info(**kwargs):
    print(kwargs) # {'name': 'Luke', 'age': 25} <-- 딕셔너리로 들어옴
    
    if 'name' in kwargs:
        print(f"Hello, {kwargs['name']}")

# 사용법: 함수 부를 때 '키=값' 형태로 전달
print_info(name="Luke", age=25, planet="Tatooine")
```

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
