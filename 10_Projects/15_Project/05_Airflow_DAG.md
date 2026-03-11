---
aliases:
tags:
  - Project
related:
  - "[[00_Project_HomePage]]"
  - "[[Airflow_Installation]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[DockerOperator_Usage]]"
  - "[[Variables_Connections]]"
  - "[[04_Spark_Batch]]"
  - "[[Airflow_Scheduling]]"
  - "[[DataFrame_Transform]]"
  - "[[Spark_Functions_Library]]"
linked:
  - file:////Users/gong/gong_study_de/project/spark_app/daily_report.py
  - file:////Users/gong/gong_study_de/project/dags/daily_report_dag.py
---
# Airflow 자동화

### 전체 구조 한눈에 보기

```text
[내 맥북]
    │
    ├─ docker-compose.yml          ← 전체 컨테이너 설정
    ├─ dags/daily_report_dag.py    ← Airflow가 읽는 DAG 파일
    ├─ spark_app/daily_report.py   ← 실제 Spark 코드
    └─ spark_drivers/*.jar         ← PostgreSQL JDBC 드라이버
```

```text
[도커 네트워크: project_data-network]
    │
    ├─ airflow-worker    ← DAG 읽고 DockerOperator 지시
    ├─ spark-master      ← Spark 클러스터 마스터
    ├─ spark-worker      ← 실제 연산 담당
    ├─ postgres          ← 데이터베이스
    └─ [새 컨테이너]     ← DockerOperator가 만들고 삭제하는 임시 컨테이너
```


## Docker 권한 설정 

Airflow가 내 맥북의 도커를 조종하려면 소켓 연결이 필요합니다.
 이게 없으면 DockerOperator 작동 안 함

```text
airflow-worker → /var/run/docker.sock → 내 맥북 도커 데몬 (이 통로로 컨테이너 생성/삭제 명령)
```

```yaml
# docker-compose.yaml 파일 내부

services:
  airflow-worker:  # (또는 x-airflow-common)
    # ... 기존 설정들 ...
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
      # 👇 [추가] Airflow가 내 컴퓨터의 Docker를 조종하기 위한 필수 설정!
      - /var/run/docker.sock:/var/run/docker.sock
```

---
## Airflow UI Variables 설정

DB 비밀번호 같은 민감 정보를 코드에 하드코딩하지 않기 위해 Airflow UI에 저장합니다.

>Airflow UI → Admin → Variables

```text
Key: daily_report_db_host / Value: project-postgres-1 
Key: daily_report_db_pwd / Value: airflow
```

```python
Variable.get("daily_report_db_host") # → "project-postgres-1"
Variable.get("daily_report_db_pwd") # → "airflow"
```

---
## DAG 실행 흐름

```scss title='DAG 실행흐름 '
① 스케줄러가 daily_report_dag 실행
        ↓
② DockerOperator가 새 컨테이너 생성
   image: project-spark-worker:latest
        ↓
③ mounts로 파일 연결
   내 맥북 spark_app/     →  컨테이너 /opt/spark/apps/
   내 맥북 spark_drivers/ →  컨테이너 /opt/spark/drivers/
        ↓
④ environment로 DB 접속 정보 주입
   DB_HOST=project-postgres-1
   DB_PASSWORD=airflow ...
        ↓
⑤ command 실행
   spark-submit
     --master spark://spark-master:7077
     --jars /opt/spark/drivers/postgresql-42.7.1.jar
     /opt/spark/apps/daily_report.py
        ↓
⑥ 작업 완료 → auto_remove=True → 컨테이너 자동 삭제
```

## 경로 관계 정리 (제일 헤맨 부분 )

```text
[내 맥북]                              [새 컨테이너]

spark_app/daily_report.py   →→→   /opt/spark/apps/daily_report.py
spark_drivers/*.jar         →→→   /opt/spark/drivers/*.jar
         (source)          bind          (target)
                           mount
```

>command는 반드시 **target 경로(컨테이너 기준)** 로 써야 합니다:!!!!

## network_mode 확인!!!!!!!!!!!!!

- network_mode='project_data-network' 없으면: 새 컨테이너 → "spark-master가 어디있지?" → ❌ 모름 새 컨테이너 → "postgres가 어디있지?" → ❌ 모름
- network_mode='project_data-network' 있으면: 새 컨테이너 → spark://spark-master:7077 → ✅ 연결됨 새 컨테이너 → postgres:5432 → ✅ 연결됨


## DAG 파일 

```python
from airflow import DAG
from airflow.providers.docker.operators.docker import DockerOperator
from datetime import datetime, timedelta
import pendulum
from docker.types import Mount
from airflow.models import Variable  # 👈 [중요] Airflow UI에서 설정한 변수를 가져오는 모듈

# 1. 공통 설정 정의
default_args = {
    'owner': 'airflow',
    'start_date': datetime(2026, 2, 17),
    'retries': 2,              # 실패 시 2번 더 재시도
    'retry_delay': timedelta(minutes=5)  # 재시도 간격 5분
}

# 2. DAG 정의
with DAG(
    dag_id='daily_report_dag',
    default_args=default_args,
    schedule_interval='@daily',  # 매일 0시 0분에 실행
    # KST(한국 시간) 기준 설정 (pendulum 사용)
    start_date=pendulum.datetime(2026, 2, 17, tz='Asia/Seoul'),
    catchup=False  # 과거 데이터 소급 적용 방지 (현재 시점부터 실행)
) as dag:
    
    # 3. Spark 작업을 실행할 Docker 컨테이너 정의
    run_spark_job = DockerOperator(
        task_id='run_spark_daily_report',
        image='project-spark-worker:latest', # 사용할 Spark 이미지
        api_version='auto',
        auto_remove=True,  # 작업 끝나면 컨테이너 자동 삭제 (깔끔하게 유지)
        
        # ✅ [핵심 1] 네트워크 설정
        # Spark Master와 Postgres가 소속된 진짜 네트워크 이름을 적어야 통신 가능
        # (확인 명령어: docker network ls)
        network_mode='project_data-network',
        
        mount_tmp_dir=False,
        
        # ✅ [핵심 2] 환경 변수 주입 (보안 강화)
        # 하드코딩 대신 Airflow Admin -> Variables에 저장된 값을 가져옴
        # .env 파일 충돌 문제를 피하는 가장 깔끔한 방법
        environment={
            'DB_HOST': Variable.get("daily_report_db_host", default_var=""), # 예: project-postgres-1
            'DB_PORT': '5432',
            'DB_NAME': 'airflow',
            'DB_USER': 'airflow',
            'DB_PASSWORD': Variable.get("daily_report_db_pwd", default_var="") # 예: airflow
        },
        
        # 4. 실행할 Spark 명령어
        # --master: 도커 네트워크 내부의 Spark Master 주소
        # --jars: JDBC 드라이버 경로 (mounts 설정과 일치해야 함)
        command="/opt/spark/bin/spark-submit --master spark://spark-master:7077 --jars /opt/spark/drivers/postgresql-42.7.1.jar /opt/spark/apps/daily_report.py",
        
        # ✅ [핵심 3] 볼륨 마운트 (파일 공유)
        # 내 컴퓨터(Host)의 파일을 컨테이너(Container) 내부 경로로 연결
        mounts=[
            Mount(
                source="/Users/gong/gong_study_de/project/spark_app", # 내 컴퓨터의 파이썬 파일 위치
                target="/opt/spark/apps",                             # 컨테이너 안에서 보일 위치
                type='bind'
            ),
            Mount(
                source="/Users/gong/gong_study_de/project/spark_drivers", # 내 컴퓨터의 jar 파일 위치
                target="/opt/spark/drivers",                              # 컨테이너 안에서 보일 위치
                type='bind'
            )
        ]
    )
```

---
## 트러블슈팅: Overwrite 설정에도 데이터가 누적되는 현상발생

### 상황 (Problem)

- `daily_report.py`에서 결과를 저장할 때 분명히 `mode="overwrite"`를 사용함.
- **기대:** 매일 실행할 때마다 "오늘의 통계"만 테이블에 남기를 원함.
- **현실:** 실행할 때마다 데이터가 계속 누적되어 숫자가 엄청나게 커짐.

```python
# 문제의 기존 코드 (전체 데이터를 다 읽어옴)
df = spark.read.jdbc(url=jdbc_url, table="user_logs", properties=db_properties)
# ↑ 어제 데이터 100개 + 오늘 데이터 100개 = 총 200개를 읽어서 덮어씌움 (결국 누적됨)
```

### 원인 (Cause)

**Overwrite의 의미 오해:** `overwrite`는 결과 테이블을 비우고 다시 쓰는 건 맞음.
하지만 **입력 데이터(`user_logs`)** 를 읽어올 때 기간 제한 없이 **"개업일부터 오늘까지의 모든 로그"** 를 다 읽어와서 집계했기 때문.
즉, "오늘치만 쓴 게 아니라, **전체 역사를 매일 다시 덮어쓰기**" 하고 있었던 것.

### 해결 (Solution): 날짜 필터링 추가

전체 데이터를 읽은 뒤, **오늘 날짜(Timestamp)** 에 해당하는 데이터만 남기고 나머지는 버리는 `filter` 로직 추가.

```python
from pyspark.sql.functions import col, current_date

# ... (중략) ...

try:
    # 1. 전체 데이터 로드
    df = spark.read.jdbc(url=jdbc_url, table="user_logs", properties=db_properties)
    
    # ✅ 2. [수정] 오늘 날짜 데이터만 필터링!
    # DB 컬럼명이 'timestamp'이므로 이를 date 타입으로 변환 후 오늘 날짜와 비교
    today_df = df.filter(col("timestamp").cast("date") == current_date())

    print(f">>> 오늘 데이터만 추출 성공! (총 {today_df.count()}건)")
    
    # 3. 집계 (이제 df가 아니라 today_df를 사용)
    report_df = (
        today_df  # 👈 여기 변경됨
        .groupBy("category")
        .agg(...)
```