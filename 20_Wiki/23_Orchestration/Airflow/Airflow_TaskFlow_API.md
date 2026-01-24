---
aliases:
  - TaskFlow
  - "@task"
  - "@dag"
  - Decorator
  - 데코레이터 패턴
tags:
  - Airflow
  - Modern
related:
  - "[[Python_Decorators]]"
  - "[[Airflow_XComs]]"
  - "[[ETL_vs_ELT]]"
---
## 개념 한 줄 요약

**TaskFlow API**는 Airflow 2.0부터 도입된 **"더 파이썬다운(Pythonic)"** DAG 작성 방식이야.
복잡한 `PythonOperator`를 정의하는 대신, 그냥 파이썬 함수 위에 **`@task`** 라는 모자(Decorator)만 씌우면 알아서 태스크가 되는 마법이지.

---
## 왜 필요한가 (Why)

**직관적임 (Intuitive):**
- 그냥 파이썬 함수 짜듯이 코딩하면 됨. 초보자도 배우기 쉬워.

**가독성 (Readability):**
- 코드가 훨씬 짧아지고 깔끔해져서 유지보수가 편해.

**데이터 전달 혁명 (Data Handling):**
- `ti.xcom_pull` 같은 복잡한 코드 없이, 함수 리턴값을 변수처럼 주고받으면 **알아서 XCom으로 처리**해줘.

---
## Practical Context (언제 쓸까?)

**Data Pipelines:**
- 데이터가 A -> B -> C 단계로 흘러가면서 변형되는 ETL 작업에 최적화되어 있어.

**Rapid Prototyping:**
- 파이썬 로직을 그대로 옮기면 되니까, 빠르게 만들고 테스트할 때 아주 좋아.

---
##  Code Core Points (Classic vs Modern)

### A. 옛날 방식 (Classic) 

`PythonOperator`를 하나하나 만들고, `xcom_pull`로 데이터를 꺼내야 했어.

```python
def _extract(**kwargs):
    ti = kwargs['ti']
    ti.xcom_push(key='data', value=100)

t1 = PythonOperator(task_id='extract', python_callable=_extract)
```

### B. 최신 방식 (TaskFlow API) 

**`@dag`** 와 **`@task`** 데코레이터만 있으면 끝!

```python
import pendulum # 1. 펜듈럼 소환!
from airflow.decorators import dag, task

@dag(
    dag_id='task_flow_api_v1', # ID는 유니크하게
    schedule='@daily',
    # 2. datetime 대신 pendulum으로 한국 시간 명시 (Best Practice)
    start_date=pendulum.datetime(2023, 1, 1, tz="Asia/Seoul"),
    catchup=False
)
def task_flow_api_dag():
    
    # [Task 1] 데이터 추출 (Dict 리턴 -> 자동으로 XCom Push)
    @task
    def extract():
        data = {'orders': [100, 200, 300]}
        print(f"Extracted data: {data}")
        return data
    
    # [Task 2] 데이터 변환 (인자로 받음 -> 자동으로 XCom Pull)
    @task
    def transform(data):
        total = sum(data['orders'])
        transformed_data = {'total_orders': total}
        print(f"Transformed data: {transformed_data}")
        return transformed_data
    
    # [Task 3] 데이터 적재
    @task
    def load(data):
        print(f"Loading data: {data}")
        
    # --- 흐름 정의 (Flow) ---
    # 변수에 담아서 넘기면 순서도 정해지고 데이터도 넘어감!
    raw_data = extract()
    processed_data = transform(raw_data)
    load(processed_data)

# DAG 등록
etl_dag = task_flow_api_dag()
```

![[스크린샷 2026-01-23 오후 6.54.26.png|300x200]]

---
## Key Features (기능 상세)

**Simplified DAG Definition:**
- DAG 자체도 `with DAG(...)` 대신 **`@dag`** 데코레이터를 쓴 함수로 정의할 수 있어. 
- 모든 설정과 의존성이 함수 하나에 예쁘게 담기지.

**Implicit XComs (암묵적 XCom):**
- **보낼 때:** 그냥 `return value` 하면 됨.
- **받을 때:** 그냥 함수 인자 `func(value)`로 넣으면 됨.
- Airflow가 뒤에서 몰래 `xcom_push`와 `pull`을 다 해주는 거야.

---
## 초보자가 자주 착각하는 포인트

1. "BashOperator는요?"
	- TaskFlow API는 주로 **Python** 태스크를 위해 만들어졌어.
	- `BashOperator` 같은 전통적인 오퍼레이터랑 섞어서 쓸 수도 있어! (`.output` 속성을 사용하면 됨)

2. "함수 밖에서 변수 쓰면?"
	- `@task` 함수 안의 코드는 워커(Worker)에서 돌지만, 함수 밖의 코드는 스케줄러에서 돌아. 
	- 전역 변수 사용은 조심해야 해.

3. Q: 마지막 줄 `etl_dag = ...` 변수명은 아무거나 써도 돼?
	- **응! 아무거나 상관없어.**  
	- 하지만 **DAG ID**랑 헷갈리면 안 돼!
	- **실무 꿀팁 (Best Practice):** 헷갈리니까 보통은 **변수 이름을 그냥 `dag`로 통일**하거나, **`dag_id`와 비슷하게** 짓는 편이야.

|**구분**|**변수 이름 (etl_dag)**|**DAG ID ('task_flow_api_v1')**|
|---|---|---|
|**정체**|파이썬 파일 안에서만 쓰는 **별명**|Airflow 전체에서 쓰는 **주민등록번호**|
|**용도**|파이썬이 객체를 담아두는 그릇|**웹 UI 리스트에 표시되는 진짜 이름**|
|**제약**|파이썬 변수 규칙만 지키면 됨|**전체 Airflow에서 유일(Unique)**해야 함!|


>이 `TaskFlow` 방식이 내부적으로 어떻게 동작하는지 궁금하면, 
>Python 기초인 **[[Python_Decorators]]** 문서를 꼭 읽어봐!