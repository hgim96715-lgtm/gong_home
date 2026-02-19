---
aliases:
  - DAG 템플릿
  - default_args
  - Pendulum
  - start_date
  - catchup
  - depends_on_past
tags:
  - Airflow
  - DAG
related:
  - "[[DAG_Concept]]"
  - "[[Airflow_DAG_Operators]]"
  - "[[Airflow_UI_Usage]]"
  - "[[Airflow_TaskGroup]]"
  - "[[Catchup_and_Backfill]]"
  - "[[Terminal_Editors]]"
  - "[[DockerOperator_Usage]]"
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
## 표준 템플릿 코드 (Copy & Paste) 

**Airflow 2.x / 3.x 최신 표준** (EmptyOperator, Pendulum, TaskFlow 고려)

```python
from airflow import DAG
from airflow.operators.empty import EmptyOperator  # 구 DummyOperator
from airflow.operators.python import PythonOperator
from datetime import timedelta
import pendulum

# 1. 공통 설정 (Config): 모든 태스크가 물려받을 속성
default_args = {
    'owner': 'data_team',           # 실명제 (에러 나면 이 팀 찾으세요)
    'retries': 3,                   # 재시도: 실패 시 3번 부활 시도
    'retry_delay': timedelta(minutes=5), # 쿨타임: 5분 쉬고 재시도, 기본값 5분
    'depends_on_past': False,       # 과거 무시: 어제 망해도 오늘은 돈다
}

# 2. DAG 정의 (Definition)
with DAG(
    dag_id='my_first_dag',          # [중요] 파일명과 일치시키는 게 국룰
    description='Airflow DAG 표준 템플릿',
    default_args=default_args,      # 위에서 만든 설정 주입
    schedule='0 9 * * *',           # 매일 아침 9시 (KST 기준)
    # [중요] 타임존은 무조건 Pendulum으로 고정!
    start_date=pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"),
    catchup=False,                  # 밀린 숙제(과거 데이터) 실행 금지
    tags=['템플릿', '기본']
) as dag:

    # 3. 로직 함수 정의
    def print_hello():
        print("Hello Airflow!")

    # 4. 태스크 정의
    t1 = PythonOperator(
        task_id='python_task',
        python_callable=print_hello
    )

    t2 = EmptyOperator(
        task_id='end_task'
    )

    # 5. 실행 순서 (Dependency)
    t1 >> t2
```

>이렇게 **딕셔너리(`default_args`)** 로 빼두면, 나중에 팀 전체 공통 설정을 만들 때 **`from my_team_config import default_args`** 처럼 다른 파일에서 불러와서 쓸 수도 있어요. (재사용성 200% 증가! )

---
## 핵심 설정값 4대장 (Deep Dive)

### ① `start_date`: 언제부터 시작해? (ft. Pendulum,Timezone(tz))

- **❌ 절대 금지:** `datetime.now()` (기준이 매번 바뀌어서 실행 안 됨)
- **⭕️ 무조건 사용:** `pendulum.datetime(..., tz="Asia/Seoul")`
	- **이유:** 한국 시간(KST)을 명시하지 않으면, 아침 9시 스케줄이 영국 시간(UTC) 기준이라 저녁 6시에 도는 대참사가 발생함.(한국 KST = UTC + 9시간)
	- `start_date`에 `tz="Asia/Seoul"`을 넣으면, CRON(`0 9 * * *`)도 알아서 한국 시간 9시로 동작한다!

### ② `catchup`: 밀린 방학 숙제 할까?

- **`True`:** 1월 1일부터 오늘까지 안 돈 거 **한꺼번에 실행.** (서버 폭발 원인 💥)
- **`False` (추천):** 과거는 잊고 **오늘 스케줄부터** 실행.

### ③ `schedule`: 언제 돌릴까?

- **Cron 식:** `'0 9 * * *'` (매일 9시 0분)
- **프리셋:** `'@daily'`, `'@hourly'`
- 💡 헷갈리면? 👉 [[Airflow_Scheduling#💡 꿀팁 자주 쓰는 Cron 프리셋 (Presets)|프리셋 모음]]

### ④ `default_args`: 귀찮은 설정 상속하기

- 태스크마다 일일이 적기 귀찮은 설정을 부모(DAG)에게 맡기는 거야.
> 💡 **Tip:** 상속 규칙이 헷갈린다면? 👉 [[Airflow_DAG_Skeleton#💡 Deep Dive `default_args`는 왜 쓸까? (상속의 마법)|상속의 마법 심화편 참고]]

| **속성**          | **설명**               | **추천 값**               |
| --------------- | -------------------- | ---------------------- |
| **owner**       | 책임자 명시               | 팀명 or 본인 ID            |
| **retries**     | **부활 횟수** (실패 시 재시도) | `3` (회)(보통 1~3회)       |
| **retry_delay** | **부활 쿨타임** (대기 시간)   | `timedelta(minutes=5)` |

>retry_delay
> **기본값:** 안 적으면 **5분(300초)** 이 적용돼.
>**실무 팁:** 기본값이 있어도 **명시적으로 적어주는 게 국룰**이야. (그래야 나중에 10분으로 늘리거나 1분으로 줄일 때 헷갈리지 않아.)

> retries랑 backfill -> [[Catchup_and_Backfill#내가 헷갈리는 것 Retries vs Backfill|retries vs backfill]] 참고 

---
### 💡 Deep Dive: `default_args`는 왜 쓸까? (상속의 마법)

`default_args`는 **"공통 설정 묶음"** 이야. 
이걸 DAG에 한 번만 등록해두면, 그 DAG 안에 있는 모든 태스크들이 이 설정을 자동으로 물려받아(상속).

#### 1. 딕셔너리 구조 뜯어보기

```python
default_args = {
    'owner': 'data_team',      # 주인: 실패 시 이메일 받을 팀
    'retries': 3,              # 재시도: 실패하면 3번 더 해봐라
    'retry_delay': timedelta(minutes=5), # 간격: 재시도할 때 5분 쉬고 해라
    'depends_on_past': False,  # 과거 의존성: 어제 망해도 오늘은 돌려라
}
```

#### 2. 우선순위 법칙 (Override Rule) ⭐️중요!

**"더 구체적인 놈이 이긴다"** 만 기억하면 돼.
1. **Level 1 (DAG 기본값):** `default_args`에 적힌 내용 (가장 약함)
2. **Level 2 (Task 개별 설정):** 오퍼레이터 안에 직접 적은 내용 (가장 쎔)

**예시 코드:**

```python
with DAG(..., default_args={'retries': 3}) as dag:
    
    # Task A: 별말 없으니까 DAG 기본값 따름 -> 재시도 3회
    task_a = EmptyOperator(task_id='A')

    # Task B: "난 특별하니까 5번 할래!" -> DAG 무시하고 재시도 5회 적용
    task_b = EmptyOperator(task_id='B', retries=5)
```

**결론:**
> 90%의 태스크가 쓸 설정은 `default_args`에 몰아넣고, 특별한 놈(Task B)만 따로 챙겨주면 코드가 훨씬 깔끔해져!


---

## 초보자가 자주 착각하는 포인트

1. **`start_date`를 `datetime.now()`로 설정하기**
    - **절대 금지!** Airflow는 `start_date` + `schedule_interval`이 지난 시점에 실행됩니다. 
    - `now()`로 하면 기준점이 계속 움직여서 영원히 실행되지 않거나 꼬입니다. 고정된 날짜를 박으세요.
        
2. **DAG 파일명과 `dag_id`를 다르게 짓기**
    - 동작은 하지만 관리하기 힘듭니다. 파일명이 `my_pipeline.py`면 DAG ID도 `my_pipeline`으로 맞추는 게 국룰입니다.

3. **"로그(`print`)가 어디에 찍히나요?"**
	- 내 터미널에 안 나옵니다. 웹 화면에서 해당 Task(네모 박스)를 클릭하고 **[Logs]** 탭을 눌러야 보입니다.

4. **"터미널에 `print` 로그가 안 보여요!"**
	- 터미널엔 안 찍혀! 웹 UI에서 **Task 클릭 -> [Logs] 탭**을 눌러야 보여.

5. "파일명을 바꿨는데 예전 DAG가 남아있어요."
	- 스케줄러가 인식하는 데 시간이 좀 걸려. 
	- 아니면 `docker-compose restart`로 재부팅 해봐.

