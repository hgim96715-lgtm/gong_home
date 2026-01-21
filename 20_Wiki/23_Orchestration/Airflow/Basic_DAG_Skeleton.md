---
aliases:
  - DAG 템플릿
  - Airflow 기본 코드
  - default_args
  - 스케줄링 설정
tags:
  - Airflow
  - 코드구현
  - DAG
  - 템플릿
related:
  - "[[DAG_Concept]]"
  - "[[DAG_Operators_Basic]]"
  - "[[Airflow_UI_Usage]]"
---
## 개념 한 줄 요약

Airflow에서 파이프라인을 만들 때 매번 복사해서 쓰는 **가장 기초적인 파이썬 코드 구조(Boilerplate Code)** 입니다.

---
## 📍 저장 위치 (가장 중요!)

* Airflow는 아무 데나 파일을 만든다고 실행해주지 않습니다.
*  반드시 **`dags` 폴더** 안에 `.py` 파일로 저장해야 스케줄러가 읽어갑니다.
*  **Docker 설치 시:** `dags/` 폴더 (예: `./dags/my_first_dag.py`)
* **Standalone 설치 시:** `~/airflow/dags/` 폴더
*  파일을 저장하고 잠시 기다리면(약 30초~1분), 웹 화면(Web UI)에 자동으로 나타납니다.
* 파일을 저장하고 웹 화면(`localhost:8080`)에서 새로고침 하세요. <- Web UI

---
## 왜 필요한가 (Why)

**문제 상황:**
DAG를 만들 때마다 `import` 구문, `default_args` 설정, `schedule_interval` 등을 일일이 타이핑하면 오타가 나기 쉽고 시간이 오래 걸립니다.

**해결책:**
검증된 '뼈대 코드'를 저장해두고, **Task(내용물)와 순서(Dependency)** 만 갈아끼우는 방식으로 개발 속도를 높입니다.

---
## 실무 맥락에서의 사용 이유

회사에서는 보통 팀 공용 **'DAG 생성기(Generator)'** 나 **'표준 템플릿'** 을 정해두고 씁니다.
"주인(Owner) 이름은 꼭 팀명으로 해라", "재시도(Retries)는 기본 3회로 해라" 같은 규칙을 `default_args`에 미리 박아두어 운영 실수를 방지합니다.

---
## 코드 핵심 포인트

1.  **`default_args`:** 모든 태스크에 공통으로 적용될 설정(주인, 재시도 횟수 등)을 딕셔너리로 만듭니다.
2.  **`with DAG(...)`:** Python의 `with` 문법을 쓰면 `dag=dag`를 매번 안 써도 돼서 깔끔합니다. (최신 권장 방식)
3.  **`schedule_interval`:** 크론 표현식(`0 9 * * *`)이나 프리셋(`@daily`)을 사용합니다.

## 상세 분석 (Copy & Paste 용)

```python
# 1. 필요한 모듈 임포트
from airflow import DAG
from airflow.operators.empty import EmptyOperator # 구 DummyOperator (이게 최신 표준!)
from airflow.operators.python import PythonOperator # 파이썬 함수를 실행하는 오퍼레이터
from datetime import datetime

# 2. 실행할 로직 정의 (Python 함수)
def print_hello():
    print('우와! Airflow다!')  # 이 메시지는 Airflow 로그에서 확인할 수 있습니다.

# 3. DAG 객체 생성 (Context Manager 방식)
with DAG(
    'my_first_dag',                          # [중요] DAG ID: Airflow 웹에서 보이는 유일한 이름
    description='나의 첫번째 Airflow DAG',      # 설명: DAG 이름 밑에 작게 나옴
    schedule='0 0 * * *',           # 스케줄: 매일 0시 0분에 실행 (Cron 표현식), 
    # [중요] schedule_interval -> schedule 로 변경됨!
    start_date=datetime(2026, 1, 21),        # 시작일: 이 날짜 이후의 스케줄부터 실행됨
    catchup=False                            # Catchup: False면 과거 안 돈거 무시, True면 과거꺼 다 돌림(주의!)
) as dag:
    
    # 4. 첫 번째 Task 정의 (PythonOperator)
    task1 = PythonOperator(
        task_id='print_hello_task',          # Task ID: 그래프 상에 표시될 이름 (DAG 안에서 유일해야 함)
        python_callable=print_hello,         # 실행할 파이썬 함수 이름 (괄호 () 빼고 이름만!)
        dag=dag                              # 소속될 DAG 명시
    )
    
    # 5. 두 번째 Task 정의 (DummyOperator)
    task2 = DummyOperator(
        task_id='dummy_task',                # 그냥 '성공'만 찍고 끝나는 태스크
        dag=dag
    )
    
    # 6. 의존성 설정 (실행 순서)
    task1 >> task2  # task1이 성공해야 -> task2가 실행된다
```

- **`schedule_interval` → `schedule`**:
    - 옛날에는 `schedule_interval`이라고 길게 썼지만, 이제는 **`schedule`** 로 통일되었습니다.
        
- **`DummyOperator` → `EmptyOperator`**:
    - '바보(Dummy)'라는 어감이 좋지 않아서, 기능에 충실한 **'비어있음(Empty)'** 으로 공식 명칭이 바뀌었습니다.


**`depends_on_past': False`**

- "어제 DAG가 실패했어도 오늘은 오늘대로 돌려라"라는 뜻입니다. 
- `True`로 하면 어제 게 성공할 때까지 오늘 게 대기합니다. (초보 때는 `False`가 정신건강에 좋습니다.)

**`catchup=False`**

- 과거 안 돈거 한꺼번에 돌릴지 여부 확인 
- `start_date`가 2026년 1월 1일이고 오늘이 1월 21일인데 `True`면, DAG를 켜자마자 **20일 치가 한꺼번에 실행**됩니다. (서버 폭발 원인 1순위)

**`t1 >> t2`**

- 비트 시프트 연산자라고 부르며, `t1.set_downstream(t2)`와 같은 뜻입니다. 
- 화살표 방향대로 흐른다고 생각하면 됩니다.

`task2 = DummyOperator`

- 실제 작업은 안 하지만, 흐름을 제어하거나 표시하기 위해 씁니다. [[DAG_Operators_Basic#DummyOperator는 왜 쓰나요?]] 참조 

---
##  작성한 DAG 실행하기 (Step-by-Step)

> 코드를 다 짰다면, 이제 Airflow에게 "일해라!"고 명령할 차례입니다. (Docker 사용자 기준)

### 1단계: 파일 저장 (Upload)

* 터미널에 `python my_dag.py`를 칠 필요가 없습니다. (어차피 로컬엔 Airflow가 없으니까요!)
*  작성한 `.py` 파일을 **`dags` 폴더** 안에 **저장(Save)** 만 하세요.
* **원리:** **스케줄러(Scheduler)** 라는 녀석이 30초~1분마다 `dags` 폴더를 감시하다가, "어? 새 파일이네?" 하고 자동으로 읽어갑니다.

### 2단계: 서버 확인 (Health Check)

터미널에서 아래 명령어로 서버가 켜져 있는지 확인합니다.

```bash
docker ps
```

- `airflow-webserver`, `airflow-scheduler`가 리스트에 보이면 **성공**입니다.
- (이미 켜져 있다면 `standalone` 명령어는 치지 마세요. 충돌납니다.)

### 3단계: 웹에서 실행 (Trigger)

1. **웹 접속:** 브라우저를 켜고 `http://localhost:8080` 접속 (ID/PW: `airflow`/`airflow`)
2. **DAG 확인:** 목록에 방금 만든 `my_first_dag`가 떴는지 확인합니다. (안 뜨면 새로고침하며 1분 대기)
3. **ON 스위치 켜기:** DAG 이름 왼쪽의 토글 스위치를 눌러 파란색(Unpause)으로 만듭니다.
4. **실행 버튼 클릭:** 맨 오른쪽 **재생 버튼(▶️)** 을 누르고 -> **Trigger DAG**를 클릭합니다.

> 보는 방법 [[Airflow_UI_Usage]] 참조

![[스크린샷 2026-01-21 오후 2.28.50.png| 600x300]]

---
## 초보자가 자주 착각하는 포인트

1. **`start_date`를 `datetime.now()`로 설정하기**
    - **절대 금지!** Airflow는 `start_date` + `schedule_interval`이 지난 시점에 실행됩니다. 
    - `now()`로 하면 기준점이 계속 움직여서 영원히 실행되지 않거나 꼬입니다. 고정된 날짜를 박으세요.
        
2. **DAG 파일명과 `dag_id`를 다르게 짓기**
    - 동작은 하지만 관리하기 힘듭니다. 파일명이 `my_pipeline.py`면 DAG ID도 `my_pipeline`으로 맞추는 게 국룰입니다.

3. **"로그(`print`)가 어디에 찍히나요?"**
	- 내 터미널에 안 나옵니다. 웹 화면에서 해당 Task(네모 박스)를 클릭하고 **[Logs]** 탭을 눌러야 보입니다.

