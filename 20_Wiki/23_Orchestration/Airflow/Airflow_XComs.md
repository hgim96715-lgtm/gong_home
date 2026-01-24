---
aliases:
  - XCom
  - Cross Communication
  - 데이터 공유
  - ti.xcom_pull
tags:
  - Airflow
  - Python
  - Metadata
related:
  - "[[Airflow_DAG_Operators]]"
  - "[[Python_Decorators]]"
  - "[[Python_JSON]]"
---
## 개념 한 줄 요약

**XCom(Cross-Communication)** 은 Airflow의 태스크(Task)들이 서로 **작은 데이터(메타데이터)** 를 주고받기 위한 우편함 시스템이야.

* **비유:** Task 1이 퇴근하면서 **포스트잇(XCom)** 에 메모를 남기고 벽(DB)에 붙여두면, Task 2가 출근해서 그 포스트잇을 떼어보는 것과 같아.

---
## 왜 필요한가 (Why)

**문제점:**
- Airflow의 태스크들은 서로 완전히 남남(격리된 프로세스)이야. 
- Task A에서 만든 변수 `result = 100`을 Task B는 절대 알 수가 없어.

**해결책:**
- "A야, 네가 작업한 결과를 Airflow DB에 저장해둬(**Push**)
- 그럼 B가 거기서 꺼내갈게(**Pull**)." 라는 약속이 필요해. 라는 약속이 필요

---
## Practical Context (언제 쓰고, 언제 안 쓰나?)

### ✅ 써야 할 때 (Metadata)

* **파일 경로 전달:** "나 전처리 끝났고, 결과 파일은 `s3://bucket/data.csv`에 저장했어."
* **상태 플래그:** "데이터 건수가 0건이라서 다음 작업 스킵해."
* **작은 변수:** 날짜, ID, 파라미터 등.

### ⛔️ 절대 쓰면 안 될 때 (Big Data) 🚨

* **Pandas DataFrame 통째로 넘기기:** 절대 금지! Airflow DB가 터져버려.
* **이유:**
    1.  XCom은 **Airflow 메타 DB(Postgres/MySQL)** 에 저장됨.
    2.  용량 제한이 아주 빡빡함. (MySQL은 고작 **64KB**!)
    3.  데이터는 반드시 **JSON으로 직렬화(Serializable)** 가능해야 함.

---
##  Code Core Points

XCom을 쓰는 방법은 **"수동(Classic)"** 과 **"자동(Modern)"** 두 가지가 있어.

### A. Classic 방식 (`push` & `pull`)

 `ti`(Task Instance) 객체를 사용해.

```python
def generate_number(ti):
    number = 50
    # 1. 보내기 (Push): key와 value로 저장
    ti.xcom_push(key='random_number', value=number)

def read_number(ti):
    # 2. 받기 (Pull): 누가(task_ids) 보냈고, 키(key)가 뭔지 알아야 함
    number = ti.xcom_pull(key='random_number', task_ids='generate_number_task')
    print(f"받은 숫자: {number}")

# Operator 정의 시
t1 = PythonOperator(
    task_id='generate_number_task',
    python_callable=generate_number
)
```

### B. Modern 방식 (TaskFlow API) - 추천!

파이썬 함수(`@task`)를 쓰면, **`return` 값이 자동으로 XCom에 Push**되고, **인자로 받으면 자동으로 Pull**이 돼.

```python
@task
def get_ip():
    return "192.168.0.1" # 자동으로 XCom에 Push됨 (Key='return_value')

@task
def print_ip(ip_address):
    print(ip_address) # 자동으로 XCom에서 Pull해옴

# DAG 흐름 정의
ip = get_ip()
print_ip(ip)
```

**심화: 한 줄로 끝내기 (Task Chaining)**
 변수 선언도 귀찮으면 **함수 안에 함수를 바로 집어넣어도 돼.**

```python
import random
import pendulum
from airflow.decorators import task

with DAG(..., start_date=pendulum.datetime(2023, 1, 1)) as dag:
    
    @task
    def generate_number():
        return random.randint(1, 100)
        
    @task    
    def read_number(number):
        print(f"Read number: {number}")
        
    #  핵심: 의존성(>>) 설정과 데이터 전달(XCom)이 이 한 줄로 끝남!
    read_number(generate_number())
```

>`read_number(generate_number())` 이 한 줄을 Airflow는 이렇게 해석합니다:
>**순서:** "아, `generate`가 먼저 실행되어야 값을 `read`한테 넘겨주겠구나. (`generate` >> `read`)"
>**데이터:** "`generate`의 리턴값을 XCom에 저장했다가, `read` 실행될 때 인자로 꽂아줘야지."

> 💡 **Tip:** `def` 함수 정의는 `with DAG(...)` 안에 넣어도 되고 밖에서 만들어도 됩니다.
> 상황에 따른 장단점이 궁금하다면? 👉 [[Python_Decorators#Advanced Tip 함수 정의 위치 (Inside vs Outside)|함수 정의 위치 심화 가이드 참고]]

![[스크린샷 2026-01-23 오후 2.24.29.png|600x140]]


---
## Detailed Analysis (저장 원리)

XCom은 마법이 아니라 **DB 테이블(`xcom` 테이블)** 에 저장되는 데이터야.

- **Key:** 데이터를 식별하는 이름 (기본값은 `return_value`)
- **Value:** 실제 데이터 (Pickle이나 JSON 형태)
- **Task ID & DAG ID:** 누가 보냈는지 식별표
- **Timestamp:** 언제 보냈는지

---
## 초보자가 자주 착각하는 포인트

1. "데이터프레임 넘겼는데 잘 되던데요?"
	- 데이터가 작을 땐(몇 줄) 되겠지만, 조금만 커져도 Airflow 웹서버가 느려지고 DB 용량 초과 에러가 날 거야.
	- **데이터는 S3/HDFS에 저장하고, XCom으로는 '경로(Path)'만 넘기는 게 국룰이야.**

2. "같은 태스크 안에서도 XCom 써야 하나요?"
	- 아니!  **"Inter-task(태스크 간)"** 통신용이야. 
	- 하나의 함수 안에서는 그냥 파이썬 변수 쓰면 돼.