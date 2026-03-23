---
aliases:
  - TaskFlow
  - "@task"
  - "@dag"
  - Decorator
  - 데코레이터 패턴
tags:
  - Airflow
related:
  - "[[Python_Decorators]]"
  - "[[Airflow_XComs]]"
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_Operators]]"
  - "[[Airflow_DAG_Skeleton]]"
---


# Airflow_TaskFlow_API — 더 파이썬다운 DAG 작성법

## 한 줄 요약

```
Airflow 2.0 부터 도입된 현대적 DAG 작성 방식
파이썬 함수 위에 @task 데코레이터만 씌우면 자동으로 태스크가 됨
xcom_push / xcom_pull 없이 함수 리턴값으로 데이터 전달
```

---

---

# 왜 필요한가?

```
Classic 방식의 문제:
  PythonOperator 를 하나하나 정의해야 함
  태스크 간 데이터 전달이 ti.xcom_pull() 로 복잡
  코드가 길고 반복적

TaskFlow API 의 해결:
  @task 데코레이터 하나로 태스크 선언 끝
  함수 리턴값 → 자동 XCom Push
  함수 인자 → 자동 XCom Pull
  파이썬 짜듯이 그대로 작성
```

---

---

# ① Classic vs TaskFlow 비교

## Classic 방식 (옛날)

```python
from airflow.operators.python import PythonOperator

def _extract(**kwargs):
    ti = kwargs['ti']                          # TaskInstance 꺼내기
    ti.xcom_push(key='data', value=100)        # 데이터 수동 push

def _transform(**kwargs):
    ti = kwargs['ti']
    data = ti.xcom_pull(key='data')            # 데이터 수동 pull
    ti.xcom_push(key='result', value=data * 2)

t1 = PythonOperator(task_id='extract',   python_callable=_extract)
t2 = PythonOperator(task_id='transform', python_callable=_transform)

t1 >> t2   # 의존성 별도 선언
```

## TaskFlow 방식 (현재)

```python
from airflow.decorators import dag, task

@dag(...)
def my_dag():

    @task
    def extract():
        return 100           # return 만 하면 자동 XCom Push

    @task
    def transform(data):     # 인자로 받으면 자동 XCom Pull
        return data * 2

    result = transform(extract())   # 순서 + 데이터 전달 한 번에
```

```
차이 한눈에:
  xcom_push  → return 으로 대체
  xcom_pull  → 함수 인자로 대체
  t1 >> t2   → result = transform(extract()) 으로 대체
```

---

---

# ② 전체 예제 — ETL 파이프라인

```python
import pendulum
from airflow.decorators import dag, task

@dag(
    dag_id    = 'etl_pipeline',
    schedule  = '@daily',
    start_date= pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"),  # ← pendulum 권장
    catchup   = False,
    tags      = ['etl'],
)
def etl_pipeline():

    @task
    def extract():
        """① 데이터 추출"""
        data = {'orders': [100, 200, 300]}
        return data                            # return → 자동 XCom Push

    @task
    def transform(data):
        """② 데이터 변환"""
        total = sum(data['orders'])
        return {'total_orders': total}         # return → 자동 XCom Push

    @task
    def load(data):
        """③ 데이터 적재"""
        print(f"적재 완료: {data}")
        # 마지막 태스크는 return 없어도 됨

    # 흐름 정의 — 변수에 담으면 순서 + 데이터 전달 동시에
    raw      = extract()
    processed= transform(raw)
    load(processed)

# ⚠️ DAG 등록 — 마지막 줄 필수!
# 이 줄이 없으면 Airflow Web UI 에 DAG 가 나타나지 않음
dag = etl_pipeline()
```
---

---

# ③ 핵심 개념

## @dag — DAG 선언

```python
@dag(
    dag_id    = 'my_dag',       # Airflow 전체에서 고유해야 함 (주민등록번호)
    schedule  = '@daily',       # 실행 주기
    start_date= pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"),
    catchup   = False,          # 과거 미실행 DAG 소급 실행 안 함
    tags      = ['train'],      # Web UI 필터링용 태그
)
def my_dag():
    ...
```

## @task — 태스크 선언

```python
@task
def my_task(input_data):
    # 로직
    return output_data   # return 이 XCom Push
```

```
@task 안에서 기억할 것:
  함수 안의 코드  → 워커(Worker) 에서 실행
  함수 밖의 코드  → 스케줄러에서 실행
  → 전역 변수 / 외부 상태에 의존하면 안 됨
```

## XCom — 태스크 간 데이터 전달

```
XCom (Cross-Communication)
  태스크 간 데이터를 주고받는 Airflow 내부 저장소

Classic:  ti.xcom_push(key='data', value=100)
          ti.xcom_pull(task_ids='extract', key='data')

TaskFlow: return 100        → 자동 push
          def task(data)    → 자동 pull

⚠️ XCom 은 작은 데이터(설정값, ID, 집계 결과 등) 에 적합
   큰 데이터(DataFrame, 파일) 는 DB / S3 에 저장하고 경로만 전달
```

## pendulum — datetime 대신 쓰는 이유

```
datetime(2026, 1, 1)
  → 타임존 정보 없음 → Airflow 가 UTC 로 처리
  → 한국 기준 스케줄이 9시간 밀릴 수 있음

pendulum.datetime(2026, 1, 1, tz="Asia/Seoul")
  → KST 명시 → 한국 시간 기준으로 정확히 실행
  → Airflow 공식 Best Practice
```

```python
import pendulum

# ✅ 권장
start_date = pendulum.datetime(2026, 1, 1, tz="Asia/Seoul")

# ❌ 타임존 없음
from datetime import datetime
start_date = datetime(2026, 1, 1)
```

---

---

# ④ 자주 묻는 것들

## 마지막 줄 변수명은 뭐든 상관없나?

```python
dag        = etl_pipeline()   # ✅ 관행적으로 dag 로 통일
etl_dag    = etl_pipeline()   # ✅ dag_id 와 비슷하게
whatever   = etl_pipeline()   # ✅ 파이썬 변수 규칙만 지키면 됨
```

|구분|변수 이름 (`dag`)|DAG ID (`'etl_pipeline'`)|
|---|---|---|
|정체|파이썬 파일 안에서만 쓰는 별명|Airflow 전체에서 쓰는 주민등록번호|
|용도|파이썬이 객체를 담는 그릇|Web UI 에 표시되는 진짜 이름|
|제약|파이썬 변수 규칙만|**전체 Airflow 에서 유일해야 함**|


## BashOperator 랑 섞어 쓸 수 있나?

```python
from airflow.operators.bash import BashOperator

@dag(...)
def mixed_dag():

    @task
    def python_task():
        return "hello"

    bash_task = BashOperator(
        task_id     = 'bash_task',
        bash_command= 'echo done',
    )

    # .output 으로 @task 결과를 BashOperator 에 전달
    python_task() >> bash_task
```

```
TaskFlow 는 Python 태스크 전용
BashOperator / EmailOperator 등 기존 오퍼레이터와 >> 로 연결 가능
```

## catchup=False 를 안 쓰면?

```
catchup=True (기본값)
  start_date 부터 지금까지 실행 안 된 DAG 전부 소급 실행
  → 처음 켰을 때 수백 개 DAG 이 한꺼번에 실행될 수 있음
  → 항상 catchup=False 명시하는 습관
```

---

---

# 치트시트

|항목|Classic|TaskFlow|
|---|---|---|
|태스크 선언|`PythonOperator(...)`|`@task`|
|데이터 전달|`ti.xcom_push / pull`|`return` / 함수 인자|
|의존성 선언|`t1 >> t2`|`b = task_b(task_a())`|
|DAG 선언|`with DAG(...):`|`@dag`|
|가독성|장황함|파이썬스러움|

> 데코레이터 개념이 낯설면 → [[Python_Decorators]] 먼저 읽기