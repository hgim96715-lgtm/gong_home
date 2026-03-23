
## 목표

```
데이터 파이프라인(DAG) 을 직접 작성하고
스케줄링 / 운영 / 트러블슈팅까지 할 수 있는 실력
```

---

---

## Level 0. 환경 세팅

| 노트                        | 핵심 개념                                            |
| ------------------------- | ------------------------------------------------ |
| [[Airflow_Installation]]  | Docker vs Standalone / pip install / standalone  |
| [[Airflow_Configuration]] | airflow.cfg / load_examples=False / 환경변수         |
| [[Airflow_UI_Usage]]      | Graph View / Log / Retry / Task Instance / Clear |


```python
⚠️ PostgreSQL 연결 에러 발생 시:
  FATAL: password authentication failed for user "airflow"
  → PostgreSQL init.sql 에 airflow DB / USER 생성 필요
  # [[PostgreSQL_Setup#Airflow DB 연결 실패 상세]] 참고

⚠️ Airflow UI 접속 불가 (admin 계정 없음):
  → airflow users create 로 admin 계정 직접 생성 필요
  # [[PostgreSQL_Setup#Step 3 — admin 계정 생성]] 참고
```

---

---

## Level 1. 핵심 개념

|노트|핵심 개념|
|---|---|
|[[Why_Airflow]]|ETL 오케스트레이션 / 스케줄링 / 의존성 관리|
|[[Airflow_Architecture]]|Webserver / Scheduler / Worker / MetaDB / Executor|
|[[Airflow_DAG_Concept]]|방향성 비순환 그래프 / Task / 의존성 / 실행 순서|
|[[Airflow_Execution_Date]]|execution_date / logical_date / 오늘 돌지만 어제 날짜|

---

---

## Level 2. DAG 작성하기

|노트|핵심 개념|
|---|---|
|[[Airflow_DAG_Skeleton]]|`with DAG(...)` 기본 골격 / dag_id / start_date / catchup|
|[[Airflow_Operators]]|PythonOperator / python_callable / op_kwargs|
|[[Airflow_TaskFlow_API]]|@task / @dag / TaskFlow API / XCom 자동화|
|[[Task_Dependencies]]|`>>` / `<<` / 리스트 / fan-out / fan-in / 병렬 실행|
|[[Airflow_Scheduling]]|cron 표현식 / @daily / catchup / timetable|
|[[Airflow_TaskGroup]]|TaskGroup / 시각적 그룹화 / prefix_group_id|

---

---

## Level 3. 데이터 통신 & 보안

| 노트                        | 핵심 개념                                     |
| ------------------------- | ----------------------------------------- |
| [[Airflow_XComs]]         | xcom_push / xcom_pull / 메타DB 저장 / 용량 제한   |
| [[Airflow_Variables_Connections]] | Variable / Connection / API Key / DB 접속정보 |
| [[Airflow_Hooks]]         | Hook / PostgresHook / S3Hook / get_conn   |

---

---

## Level 4. 워크플로우 제어

|노트|핵심 개념|
|---|---|
|[[Catchup_and_Backfill]]|catchup=True / backfill / 과거 데이터 재처리|
|[[Branching]]|BranchPythonOperator / task_id 반환 / 조건 분기|
|[[Trigger_Rules]]|all_success / all_done / one_failed / none_failed|
|[[Airflow_Sensors]]|FileSensor / HttpSensor / poke_interval / timeout|

---

---

## Level 5. 운영 & 트러블슈팅

|노트|핵심 개념|
|---|---|
|[[Airflow_CLI]]|dags list / tasks test / db init|
|[[Common_Airflow_Errors]]|return code 1 / ImportError / Zombie Task|
|[[Airflow_Best_Practices]]|멱등성 / top-level import 금지 / 경량 태스크|

---

---

## Level 6. 외부 연동

|노트|핵심 개념|
|---|---|
|[[Airflow_Providers]]|apache-airflow-providers / AWS / GCP / DB 연결|
|[[Airflow_Spark_Integration]]|SparkSubmitOperator / spark_conn_id / 자동화|
|[[MinIO_Concept]]|오브젝트 스토리지 / S3 호환 / 로컬 환경 S3 대체|
|[[MinIO_Integration]]|Spark + MinIO / S3A / hadoop-aws|

---

---

## 프로젝트 적용

|노트|설명|
|---|---|
|[[06_Hospital_Airflow]]|Hospital 프로젝트 DAG (er_hospitals / er_hourly_stats)|

---

---

