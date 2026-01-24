---
aliases:
  - Airflow Scheduling
  - Data-aware Scheduling
  - Datasets
  - Catchup
  - Backfill
  - Cron
tags:
  - Airflow
  - Dataset
  - schedule
  - cron
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Linux_Scheduling_Crontab]]"
---
## 개념 한 줄 요약

**Airflow 스케줄링**은 DAG를 **"언제"** 실행할지 결정하는 규칙이야. 
과거에는 **"시간(Time-based)"** 만 있었지만, 이제는 **"데이터(Data-aware)"** 가 업데이트되면 실행하는 방식이 추가되었어.

---
## 왜 필요한가 (Why)

**기존의 문제점 (Time-based):**
* "매일 아침 9시에 실행해"라고 했는데, 앞단의 데이터 생성 작업이 9시 5분에 끝나면? -> **데이터가 없는 채로 돌다가 에러 남.**
* 이걸 막으려 `ExternalTaskSensor`를 썼는데 코드가 복잡하고 리소스를 계속 잡아먹음.

 **해결책 (Data-aware):**
    * **Dataset** 개념 도입. "시간 상관없이, **이 데이터가 만들어지면** 그때 실행해!"가 가능해짐.
    * 이제 파이프라인 간의 의존성을 훨씬 직관적으로 연결할 수 있어.

---
## Practical Context (실무 활용)

현업에서는 이 두 가지 방식을 상황에 맞춰 섞어 써.

1.  **Time-based (Cron):** 정기 리포트, 매일 자정 로그 백업 등 **고정된 시간**이 중요할 때.
2.  **Data-aware (Dataset):** 머신러닝 파이프라인(전처리 끝나면 -> 학습), 데이터 웨어하우스 적재(소스 데이터 수집 끝나면 -> 적재).

---
##  Code Core Points & Changes

**⚠️ 중요 변경 사항 (Airflow 2.4+):**
- 과거의 `schedule_interval` 파라미터가 **`schedule`** 로 통합되었어. 
- 이제 여기에 Cron 식을 넣을 수도 있고, Dataset 객체를 넣을 수도 있어.

### A. Time-based Scheduling (Cron)

```python
from airflow import DAG
from datetime import datetime

with DAG(
    dag_id="time_based_dag",
    start_date=datetime(2023, 1, 1),
    schedule="0 9 * * *",  # (구) schedule_interval. 매일 아침 9시
    catchup=False          # 밀린 스케줄 실행 안 함 (중요!)
) as dag:
    ...
```

### 💡 꿀팁: 자주 쓰는 Cron 프리셋 (Presets)

매번 `0 0 * * *` 처럼 숫자를 나열하기 힘들다면, Airflow가 제공하는 예약어를 쓰세요. 
가독성이 훨씬 좋아집니다.

| Preset | 의미 (Meaning) | Cron 등가식 | 비고 |
| :--- | :--- | :--- | :--- |
| `None` | **스케줄 없음** | - | 오직 외부 트리거(UI 버튼, API)로만 실행될 때 사용 |
| `@once` | **딱 한 번만 실행** | - | 디버깅용이나 1회성 작업에 유용 |
| `@hourly` | 매시간 0분에 실행 | `0 * * * *` | - |
| `@daily` | 매일 자정(00:00)에 실행 | `0 0 * * *` | 가장 많이 씀! |
| `@weekly` | 매주 일요일 자정에 실행 | `0 0 * * 0` | - |
| `@monthly` | 매월 1일 자정에 실행 | `0 0 1 * *` | 월간 리포트용 |
| `@yearly` | 매년 1월 1일 자정에 실행 | `0 0 1 1 *` | - |

**사용 예시:**

```python
with DAG(
    dag_id="my_daily_dag",
    schedule="@daily",  # "0 0 * * *" 대신 이렇게 쓰면 됨!
    ...
)
```


### B. Data-aware Scheduling (Dataset)

**Producer(생산자)와 Consumer(소비자)** 구조로 나뉨.

```python
from airflow import DAG
from airflow.datasets import Dataset
from airflow.operators.bash import BashOperator

# 1. 데이터셋 정의 (URI 형식의 문자열)
# 실제 파일 경로가 아니라 '논리적인 이름표'라고 생각하면 됨.
# 단, Airflow가 “식별자(URI/절대경로)”로 인식할 수 있는 형태여야 한다 my_data (x)
my_data = Dataset("s3://bucket/my_data.csv")

# 2. Producer DAG (데이터를 만드는 놈)
with DAG(dag_id="producer_dag", schedule="@daily", ...):
    task1 = BashOperator(
        task_id="create_data",
        bash_command="echo 'data created' >> /tmp/data.csv",
        outlets=[my_data]  # 이 태스크가 끝나면 my_data를 업데이트했다고 알림!
    )

# 3. Consumer DAG (데이터를 쓰는 놈)
with DAG(
    dag_id="consumer_dag", 
    schedule=[my_data],  # my_data가 업데이트되면 즉시 실행! (시간 설정 X)
    ...
):
    task2 = BashOperator(...)
```

- **Dataset 정의:** `s3://` 같은 **URI 형식**이나 `/tmp/...` 같은 **절대 경로**를 주로 사용해요. 
    - *주의:* `airflow://` 같은 예약어는 못 쓰고, **정규표현식(Regex)도 지원하지 않아요.** (문자열이 100% 일치해야 함)(`*.csv` 이런 거 안 됨!)
- **`outlets` (배출구):** 이 태스크가 끝나면 "나 이 데이터 업데이트했어!"라고 Airflow에 신고하는 선언이에요.
- **`schedule` (구독):** 리스트 `[...]` 안에 데이터셋을 넣어두면, 그 데이터가 업데이트될 때마다 DAG가 자동으로 실행돼요.

---
## 코드 분석

### 1) Dataset의 한계 (Limitations)

이 기능이 만능은 아니야. 주의할 점들이 있어.

- **내용 검증 안 함:** Airflow는 실제 파일이 `s3`에 생겼는지 확인하지 않아. 
- 그냥 `outlets`에 등록된 태스크가 **"성공(Success)"** 하면 업데이트됐다고 칠 뿐이야.
- **스케줄 혼용 불가:** `schedule=[my_data]`와 `schedule="0 9 * * *"`를 동시에 쓸 수 없어. 
- (데이터도 들어오고 시간도 9시여야 실행? -> 이거 안 됨).
- **외부 툴 감지 불가:** Airflow 외부(예: 람다 함수, 스파크 잡)가 S3에 파일을 몰래 넣어도 Airflow는 몰라. 
- 오직 Airflow Task를 통해서만 트리거됨.

### 2) Catchup과 Backfill

Time-based 스케줄링에서 초보자가 제일 많이 실수하는 부분.

- **Catchup=True (Default):** 
- `start_date`가 1년 전이면, 오늘 DAG를 켜는 순간 **지난 1년 치(365번)가 한꺼번에 실행됨.**
- (서버 폭발의 주범 💥)

- **Catchup=False:** 과거는 잊고, **"다음 예정된 스케줄"** 부터 실행함. 
- 개발할 땐 무조건 `False`로 두는 게 정신건강에 좋아.

---
## 초보자가 자주 착각하는 포인트

1.  **URI는 실제 경로여야 한다?**
    - 아니! `Dataset("s3://my-bucket/file.csv")`라고 썼다고 해서 Airflow가 S3에 접속해보는 게 아님. 
    - 그냥 **"이런 이름표"** 를 공유하는 것뿐이야. 
    - `Dataset("my_team_table_A")` 처럼 써도 동작해.

2. **`schedule_interval` 쓰면 안 되나?**
    - 여전히 작동은 하지만 `Deprecated` 경고가 뜰 거야. 
    - 최신 버전(2.4 이상)을 쓴다면 **`schedule`** 파라미터를 쓰는 습관을 들이자.

3.  **데이터가 여러 개면?**
    - `schedule=[dataset1, dataset2]` 처럼 리스트로 넣으면 됨. 
    - 이 경우 **두 데이터셋이 모두 업데이트되어야** DAG가 실행됨 (AND 조건).