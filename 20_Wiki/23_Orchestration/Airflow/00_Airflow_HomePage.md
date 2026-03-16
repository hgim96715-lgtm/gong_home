## 개념 한 줄 요약

복잡한 데이터 파이프라인(DAG) 관리 기술을 **'개념(What) → 작성(How) → 운영(Run) → 트러블슈팅(Fix)'** 순서로 정리하여, 전체 흐름을 조망하는 **관제 센터(Control Tower) 노트**입니다.

---

## 0. 시작하기 (Getting Started)

> "이론 공부 전, 내 컴퓨터에 Airflow를 띄워봅시다."

- [[Airflow_Installation]] : Docker vs Standalone 설치 가이드 (`Docker Compose`, `pip install`, `standalone`, `맥북 유저 필독`)
- [[Airflow_Configuration]] : 초기 설정 꿀팁 (`airflow.cfg`, `예제 DAG 제거`, `load_examples = False`, `환경변수`)
- [[Docker_Host_Access]] : Docker 컨테이너에서 내 컴퓨터(DB) 접속하는 법 (`host.docker.internal`, `네트워크 브릿지`, `포트 바인딩`)
- [[Airflow_UI_Usage]] : 로그 확인 & 재시도 버튼 위치 완벽 정리 (`Graph View`, `Log`, `Retry`, `Task Instance`, `Clear`)

---

## 핵심 철학 (Core Concepts)

> "Airflow가 돌아가는 원리 이해하기 (면접 단골 질문!)"

- [[Why_Airflow]] : 데이터 파이프라인과 Airflow 사용 이유 (`ETL`, `오케스트레이션`, `스케줄링`, `의존성 관리`)
- [[ETL_vs_ELT]] : 요즘 왜 ETL 대신 ELT를 많이 쓸까? (`ETL`, `ELT`, `Modern Data Stack`, `dbt`, `BigQuery`)
- [[Airflow_Architecture]] : 웹서버, 스케줄러, 워커, 메타DB의 4박자 관계 (`Webserver`, `Scheduler`, `Worker`, `MetaDB`, `Executor`)
- [[DAG_Concept]] : DAG가 도대체 뭔가요? (`방향성 비순환 그래프`, `Task`, `의존성`, `실행 순서`, `Directed Acyclic Graph`)
- [[Airflow_Providers]] : 외부 서비스와 연동하기 위한 확장팩 (`apache-airflow-providers`, `AWS`, `GCP`, `Slack`, `DB 연결`)
- [[Airflow_Execution_Date]] : 가장 헷갈리는 '실행 기준일' 개념 잡기 (`execution_date`, `logical_date`, `data_interval_start`, `오늘 돌지만 어제 날짜`)

---

## 1. DAG 작성하기 (Building Pipelines)

> "레고 블록 조립하듯 워크플로우 짜기"

- [[Airflow_DAG_Skeleton]] : `with DAG(...)` 기본 골격 템플릿 (`dag_id`, `start_date`, `schedule_interval`, `catchup`, `default_args`)
- [[Airflow_Operators]] : PythonOperator 완벽 정리 (`PythonOperator`, `python_callable`, `op_kwargs`, `Classic Way`)
- [[DockerOperator_Usage]] : 컨테이너로 태스크 격리해서 실행하기 (`DockerOperator`, `image`, `command`, `volumes`, `환경 격리`)
- [[Airflow_TaskFlow_API]] : `@task` 데코레이터로 파이썬 함수처럼 짜기 (`@task`, `@dag`, `TaskFlow API`, `Modern Way`, `XCom 자동화`)
- [[Task_Dependencies]] : 순서 정하기 (`>>`, `<<`, `[]` 리스트`, `fan-out`, `fan-in`, `병렬 실행`)
- [[Dynamic_Tasks]] : `for`문으로 태스크 대량 생산하기 (`for 루프`, `Classic Pattern`, `동적 생성`, `task_id 변수화`)
- [[Dynamic_Task_Mapping]] : `expand` 함수로 우아하게 매핑하기 (`expand`, `partial`, `Modern Pattern`, `병렬 매핑`)
- [[Airflow_Scheduling]] : 스케줄링 정복 (`cron 표현식`, `@daily`, `Dataset`, `catchup`, `timetable`)
- [[Airflow_TaskGroup]] : 태스크들을 폴더처럼 묶어서 정리하기 (`TaskGroup`, `with TaskGroup`, `시각적 그룹화`, `prefix_group_id`)

---

## 2. 데이터 통신 및 보안 (Data Communication & Security)

> "태스크끼리 데이터를 넘겨주고, 비밀번호는 안전하게 관리해요."

- [[Airflow_XComs]] : 작고 가벼운 데이터 토스하기 (`xcom_push`, `xcom_pull`, `메타DB 저장`, `용량 제한 주의`)
- [[Variables_Connections]] : 비밀번호와 접속 정보 안전하게 관리하기 (`Variable`, `Connection`, `API Key`, `DB접속정보`, `UI에서 등록`)
- [[Airflow_Params]] : DAG 실행 시 설정값 주입하기 (`Params`, `trigger with config`, `런타임 설정`, `JSON 입력`)
- [[Airflow_Hooks]] : 직접 외부 시스템 문 따고 들어가기 (`Hook`, `PostgresHook`, `S3Hook`, `get_conn`, `Operator 내부 원리`)

---

## 3. 워크플로우 제어 및 확장 (Workflow Control & Extension)

> "단순 실행을 넘어, 분기하고 기다리고 시간을 조종하는 고수의 기술."

- [[Airflow_CLI]] : 마우스 대신 키보드로 Airflow 조종하기 (`airflow dags list`, `airflow tasks test`, `airflow db init`, `CLI 고수`)
- [[Catchup_and_Backfill]] : 과거 데이터 재처리하기 (`catchup=True`, `backfill`, `start_date`, `과거 1년치 재처리`)
- [[Branching]] : 조건에 따라 A경로 혹은 B경로로 분기하기 (`BranchPythonOperator`, `task_id 반환`, `skip`, `조건 분기`)
- [[Trigger_Rules]] : "앞에 놈 망해도 나는 실행할래" (`trigger_rule`, `all_success`, `all_done`, `one_failed`, `none_failed`)
- [[Airflow_Sensors]] : "파일 들어올 때까지 기다려!" (`FileSensor`, `HttpSensor`, `poke_interval`, `timeout`, `외부 이벤트 감지`)
- [[Airflow_REST_API]] : 파이썬/Curl로 Airflow 원격 조종하기 (`REST API`, `DAG trigger`, `Bearer Token`, `외부 트리거`)

---

## 트러블슈팅 & 운영 (Ops & Debugging)

> "만든 파이프라인을 건강하게 돌리고, 아프면 고쳐줘요."

- [[Common_Airflow_Errors]] : 자주 보는 에러 모음 (`return code 1`, `ImportError`, `Zombie Task`, `에러 로그 읽기`)
- [[Airflow_Best_Practices]] : 실무에서 DAG 짤 때 지켜야 할 십계명 (`멱등성`, `top-level import 금지`, `경량 태스크`, `재시도 설정`)
- [[Airflow_Flower]] : 워커(Worker) 상태 감시하기 (`Flower`, `Celery`, `Worker 상태`, `큐 모니터링`, `CCTV`)
- [[Airflow_Queues_Scaling]] : 큐 분리와 성능 튜닝 (`Parallelism`, `Concurrency`, `Queue`, `Worker 수`, `CeleryExecutor`)

---

## 인프라와 외부 연동 (Infrastructure)

> "Airflow 혼자서는 반쪽짜리야. 저장소와 친구들을 연결해 보자."

- [[MinIO_Concept]] : AWS S3를 내 컴퓨터에! (`오브젝트 스토리지`, `S3 호환`, `버킷`, `로컬 환경 S3 대체`)
- [[MinIO_Docker_Setup]] : Docker Compose에 MinIO 추가하기 (`docker-compose.yml`, `포트 설정`, `ACCESS_KEY`, `SECRET_KEY`)
- [[Docker_Compose_Architecture_MinIO_Spark]] : 왜 MinIO는 합치고 Spark는 나눌까? (`최종 아키텍처`, `네트워크 분리`, `서비스 의존성`) 

---

## 대용량 데이터 처리 (Big Data Processing)

> "엑셀로는 안 열리는 GB, TB 단위 데이터를 요리해 보자."

- [[Apache_Spark_Concept]] : 하둡보다 100배 빠른 분산 처리 엔진 (`RDD`, `DataFrame`, `In-Memory`, `Driver`, `Executor`, `분산 처리`)
- [[Apache_Spark_Setup]] : Docker로 내 컴퓨터에 Spark 클러스터 구축하기 (`Master`, `Worker`, `spark-submit`, `docker-compose`)
- [[MinIO_Integration]] : Spark(연산)와 MinIO(저장)를 S3 프로토콜로 연결하기 (`S3A`, `hadoop-aws`, `spark.jars`, `aws.endpoint`)
- [[Airflow_Spark_Integration]] : Airflow로 Spark 작업 자동화하기 (`SparkSubmitOperator`, `spark_conn_id`, `application`, `자동화`)
- [[Airflow_Spark_MinIO_Pipeline_Build]] : 실전 문제 해결서 (`에러 로그`, `트러블슈팅`, `실전 경험`, `나중에 에러 나면 보는 곳`)