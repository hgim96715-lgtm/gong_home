---
aliases:
  - BranchPythonOperator
  - 분기 처리
  - 조건문
  - If-Else
  - Branching
  - 문자열 리턴
tags:
  - Airflow
  - Logic
  - ControlFlow
  - Python
  - DAG
related:
  - "[[Airflow_XComs]]"
  - "[[Airflow_DAG_Operators]]"
---
## 개념 한 줄 요약

**BranchPythonOperator**는 DAG 안의 **"신호등(Traffic Controller)"** 이야.
파이썬 함수를 실행해서 결과에 따라 **"A 길로 갈지, B 길로 갈지"** 를 동적으로 결정해 주는 오퍼레이터지.

---
## 왜 필요한가 (Why)

**문제점:**
- 기본적으로 Airflow 태스크는 일직선(`>>`)으로만 흘러가. 
- "주말에는 주식 시장이 닫히니까 수집하지 마" 같은 로직을 `BashOperator`만으로는 짜기 힘들어.

**해결책:**
- "지금이 주말이야?"라고 물어보고, `True`면 `skip_task`로, `False`면 `crawl_task`로 보내는 **제어 흐름(Control Flow)** 이 필요해.

---
## Practical Context (실무 활용)

1.  **데이터 품질 검증:** 전처리한 데이터 개수가 0개면? ➡ `Alert` 보내고 종료. 정상이면? ➡ `Load` 태스크로 이동.
2.  **요일별 로직:** 평일이면 `Stock_Market` 태스크 실행, 주말이면 `Crypto_Market` 태스크 실행.
3.  **짝수/홀수 분기:** 특정 시간에 따라 다른 로직을 태우고 싶을 때.

---
##  Code Core Points

이 코드의 핵심은 **"함수가 `task_id` 문자열을 리턴한다"** 는 점이야.

```python
import pendulum
from time import time
from airflow import DAG
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import BranchPythonOperator

# 1. 분기 로직을 담은 파이썬 함수
def decide_which_path():
    # 현재 시간이 짝수면 -> even_path_task로 가라!
    if int(time()) % 2 == 0:
        print(f"지금 시간은 짝수입니다. {int(time())}")
        return "even_path_task"  # ⭐️ 중요: 다음 실행할 태스크의 task_id를 문자열로 리턴
    
    # 홀수면 -> odd_path_task로 가라!
    else:
        print(f"지금 시간은 홀수입니다. {int(time())}")
        return "odd_path_task"
    
with DAG(
    dag_id='branch_dag',
    start_date=pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"),
    schedule='@hourly',
    catchup=False
) as dag:
    
    # 2. BranchPythonOperator 정의
    branch_task = BranchPythonOperator(
        task_id='branching_task',
        python_callable=decide_which_path 
    )
    
    # 3. 갈림길에 있는 태스크들
    even_path_task = EmptyOperator(task_id='even_path_task')
    odd_path_task = EmptyOperator(task_id='odd_path_task')
    
    # 4. 의존성 설정 (중요: 둘 다 연결해둬야 함)
    # branch_task가 실행되면, 리턴된 놈만 실행되고 나머지는 Skip됨
    branch_task >> even_path_task
    branch_task >> odd_path_task
```

---
## Detailed Analysis (작동 원리)

**If-Then Control Flow**가 작동하는 방식

1. **Evaluate:** `decide_which_path` 함수가 실행됨.
2. **Return:** 함수가 `"even_path_task"`라는 **ID**를 뱉어냄.
3. **Execute:** Airflow는 `task_id`가 일치하는 `even_path_task`를 찾아서 실행함.
4. **Skip:** 선택받지 못한 `odd_path_task`는 자동으로 **Skipped (분홍색)** 상태로 변해서 실행되지 않음.

![[스크린샷 2026-01-23 오후 2.51.29.png|500x240]]

----
### 💡 심화: 왜 Branch만 '문자열 ID'를 리턴할까?

일반 함수와 브랜치 함수는 **리턴값의 용도**가 완전히 달라!

| 구분 | **BranchPythonOperator** (신호등) 🚦 | **PythonOperator** (일꾼) 👷 |
| :--- | :--- | :--- |
| **목적** | "다음에 **어디로** 갈지 정해줄게." | "내가 **작업한 결과물(데이터)**을 줄게." |
| **리턴해야 할 것** | **Task ID (문자열)** | **Data (숫자, 문자, 리스트 등)** |
| **처리 방식** | Airflow가 이 ID를 찾아서 **그 태스크를 실행**시킴. | Airflow가 이 값을 **XCom(우편함)**에 저장함. |
| **비유** | "다음 역은 **'강남역'** 입니다." (위치 알림) | "여기 **'짜장면'** 배달 왔습니다." (물건 전달) |

**1. BranchPythonOperator (길 안내)**

```python
def branch_func():
    # "task_a"라는 이름표(ID)를 줘야 거기로 감!
    return "task_a"
```

**2. PythonOperator (데이터 생산)**

```python
def work_func():
    # 여기서 "task_a"를 리턴하면? 
    # 그냥 "task_a"라는 글자가 XCom에 저장될 뿐, 이동하지 않음!
    return {"count": 100}
```

**결론:**
- **길을 정할 때(Branch):** 무조건 **Task ID(문자열)** 리턴
- **값을 넘길 때(General):** 그냥 **데이터(객체)** 리턴

>**Branching:** "야, 너 저기 **'even_path_task'** 간판 있는 방으로 가!" 👉 **방 이름(String)** 이 필요함.
>**Normal:** "야, 내가 만든 **'객체(Object)'** 받아라!" 👉 **물건(Object)** 이 필요함.

---
## 초보자가 자주 착각하는 포인트

1. "객체를 리턴하면 안 되나요?"
	- 절대 안 됨! `return even_path_task` (객체) ❌
	-  무조건 `return "even_path_task"` (문자열 ID) ⭕️

2. "여러 개 실행할 수는 없나요?"
	- 가능해! 리스트로 리턴하면 돼. `return ["task_a", "task_b"]`라고 하면 두 개 다 실행돼.
	- 이 기능은 나중에 **Airflow 2.0 TaskFlow API(`@task.branch`)** 를 배우면 더 쉽게 쓸 수 있어.
	- 하지만 기본 원리는 이 `BranchPythonOperator`가 근본이야.

3. **"선택 안 된 태스크 뒤에 있는 애들은 어떻게 되나요?"**
	- 줄줄이 소세지처럼 다 **Skip** 돼. 
	- (이걸 막으려면 `Trigger Rule`이라는 걸 건드려야 하는데 나중에 배우자.)

