---
aliases:
  - Airflow Operator
  - BashOperator
  - PythonOperator
  - 오퍼레이터 종류
  - Sensor
tags:
  - Airflow
  - DAG
  - 코드구현
  - Operator
related:
  - "[[Airflow_Architecture(아키텍처)]]"
  - "[[Airflow_DAG_Skeleton]]"
---

## 개념 한 줄 요약

DAG(워크플로우)라는 공장에서 실제로 작업을 수행하는 **'작업자(Worker)를 정의하는 템플릿(Class)'** 입니다. (파이썬, 셸 스크립트, SQL 등 무엇을 실행할지 결정)

---
## 왜 필요한가 (Why)

**문제 상황:**
Airflow가 "작업을 실행해"라고 했을 때, 그 작업이 셸 명령어를 치는 건지, 파이썬 함수를 부르는 건지, 아니면 AWS S3에 파일을 올리는 건지 구체적인 **'행동 지침'** 이 없으면 아무것도 할 수 없습니다.

**해결책:**
Operator는 미리 만들어진 도구 상자입니다.
* "터미널 명령어를 쓰고 싶어?" -> `BashOperator`를 가져다 씀
* "파이썬 함수를 돌리고 싶어?" -> `PythonOperator`를 가져다 씀
* 이렇게 **미리 정의된 클래스(Pre-defined components)** 를 조립만 하면 복잡한 기능을 쉽게 구현할 수 있습니다.

---
## 실무 맥락에서의 사용 이유

실무 DAG의 90%는 `PythonOperator`, `BashOperator`, `KubernetesPodOperator` 3가지로 이루어집니다.
특히 데이터 엔지니어는 **"내가 직접 파이썬으로 ETL을 짤 것인가(PythonOperator)"** 아니면 **"이미 짜여진 dbt나 Spark를 실행만 시킬 것인가(BashOperator)"** 를 매번 결정해야 합니다. 이 Operator들의 특성을 정확히 알아야 적재적소에 쓸 수 있습니다.

---
## Operator의 3가지 종류 (핵심 분류)

기능에 따라 크게 3가지로 나뉩니다.

1.  **Action Operator (행동대장):**
    * 실제 연산이나 명령을 수행합니다.
    * 예: `BashOperator` (셸 스크립트 실행), `PythonOperator` (파이썬 함수 실행)

2.  **Transfer Operator (배달부):**
    * 데이터를 A에서 B로 옮깁니다.
    * 예: `S3FileTransferOperator` (S3로 파일 업로드)

3.  **Sensor Operator (감시자):**
    * 특정 조건이 만족될 때까지 **기다립니다.** (가장 중요!)
    * 예: `FileSensor` (파일이 도착했나?), `TimeSensor` (특정 시간이 됐나?), `HdfsSensor`

4.  **Utility Operator (도우미):**
    * 실제 작업은 안 하지만, 흐름을 제어하거나 표시하기 위해 씁니다.
    * 예: `DummyOperator` (아무것도 안 함, 표지판 역할)

---
### EmptyOperator(구:DummyOperator)는 왜 쓰나요? 

- **용도 1: Start/End 표지판**
    - 복잡한 DAG에서 어디가 시작이고 끝인지 한눈에 보기 위해 관습적으로 맨 앞과 맨 뒤에 둡니다.

- **용도 2: 의존성 모으기 (Fan-in/Fan-out)**
    
    - **상황:** 작업 A, B, C가 다 끝나면 작업 D, E, F를 실행해야 한다고 칩시다.
    - **Dummy 없을 때:** 선을 일일이 다 연결해야 합니다. (A>>D, A>>E, A>>F, B>>D... 총 9개 선) -> 그래프가 지저분해짐.
    - **Dummy 있을 때:** `[A, B, C] >> dummy >> [D, E, F]` (중간에 점 하나만 거치면 됨) -> 그래프가 훨씬 깔끔해짐.

- **참고:** 최신 Airflow 버전(2.x 후반)에서는 `EmptyOperator`라는 이름으로 바뀌고 있지만, 실무에서는 여전히 `DummyOperator`라고 많이 부릅니다.
- 바보(Dummy)'라는 어감이 좋지 않아서, 기능에 충실한 **'비어있음(Empty)'** 으로 공식 명칭이 바뀌었습니다.

---
## 코드 핵심 포인트

1.  **Task ID 유일성:** 모든 Operator는 `task_id`를 가져야 하며, DAG 안에서 절대 중복되면 안 됩니다.
2.  **호출 방식:**
    * Bash는 `{python}bash_command="명령어"` 형태로 문자열을 넘깁니다.
    * Python은 `{python}python_callable=함수이름` 형태로 **함수 자체(객체)** 를 넘깁니다. (함수 뒤에 `()`를 붙이면 안 됨!)

---
## 상세 분석 (Line-by-Line)

```python
# 1. BashOperator 정의 (터미널 명령어 실행)
task1 = BashOperator(
    task_id='print_date',       # 1-1. Airflow UI에 표시될 작업 이름
    bash_command='date',        # 1-2. 실제로 터미널에 칠 명령어 ('date'는 현재 날짜 출력)
    dag=dag                     # 1-3. 이 작업이 소속될 DAG 지정
)

# 2. 파이썬 함수 정의 (PythonOperator가 실행할 로직)
def python_function():
    print("Hello, Airflow!")

# 3. PythonOperator 정의 (파이썬 함수 실행)
task2 = PythonOperator(
    task_id='python_task',            # 3-1. 작업 이름 (task1과 달라야 함)
    python_callable=python_function,  # 3-2. 실행할 함수 이름 (괄호 () 없음 주의!)
    dag=dag
)
# 0. 시작점과 끝점 만들기 (EmptyOperator 활용) 
start = EmptyOperator(task_id='start', dag=dag) 
end = EmptyOperator(task_id='end', dag=dag)

# 4. 의존성 설정 (DummyOperator로 깔끔하게 묶기)
# start -> [task1, task2] -> end
# task1과 task2가 동시에 시작되고, 둘 다 끝나야 end가 실행됨
start >> [task1, task2] >> end
```

**`task_id='print_date'`**
: 나중에 에러가 났을 때 로그에서 "print_date 작업이 실패했습니다"라고 찾게 되는 이름표입니다.

**`bash_command='date'`**: 리눅스 터미널에서 `date`라고 치는 것과 똑같은 효과를 냅니다.

**`python_callable=python_function`**:
- 🚨 **주의:** `python_function()`이라고 쓰면 안 됩니다! 그러면 DAG가 로딩될 때 함수가 실행되어 버립니다.

**`task1 >> task2`**: 이것이 바로 DAG(방향성 그래프)를 만드는 핵심 문법입니다. 화살표 방향대로 실행됩니다.


---
## 초보자가 자주 착각하는 포인트

1. **Sensor는 그냥 멈춰있는 것이다?**
    - 아닙니다. Sensor도 Operator의 일종이라 **계속 실행되면서(Running)** 확인합니다.
    - "파일 왔어?" (아니) -> 30초 대기 -> "파일 왔어?" (아니) -> ...
    - 그래서 Sensor를 잘못 쓰면 워커 슬롯을 다 차지해서 다른 작업이 못 도는 **'Deadlock'** 이 발생할 수 있습니다. (고급 주제)

2. **`dag=dag`는 꼭 써야 하나요?**
    - `with DAG(...) as dag:` 문법(Context Manager)을 쓰면 생략 가능합니다. 요즘은 생략하는 추세입니다.
    - 하지만  `dag = DAG(...)` 변수 할당 방식이면 꼭 써줘야 합니다.



