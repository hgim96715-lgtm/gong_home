
## 개념 한 줄 요약

복잡한 데이터 파이프라인(DAG) 관리 기술을 **'개념(What) -> 작성(How) -> 운영(Run) -> 트러블슈팅(Fix)'** 순서로 정리하여, 전체 흐름을 조망하는 **관제 센터(Control Tower) 노트**입니다.

---

## 0. 시작하기 (Getting Started)

> "이론 공부 전, 내 컴퓨터에 Airflow를 띄워봅시다."

- [[Airflow_Installation]] : Docker vs Standalone 설치 가이드 (맥북 유저 필독!) 
- [[Airflow_Configuration]] : 초기 설정 꿀팁 (지저분한 예제 DAG 싹 지우기)
- [[Docker_Host_Access]] : Docker 컨테이너에서 내 컴퓨터(DB) 접속하는 법 (**가장 중요!**)
- [[Airflow_UI_Usage]] : 로그 확인 & 재시도 버튼 위치 완벽 정리

## 핵심 철학 (Core Concepts)

> "Airflow가 돌아가는 원리 이해하기 (면접 단골 질문!)"

- [[Why_Airflow]] : 데이터 파이프라인과 Airflow 사용 이유 (ETL, 오케스트레이션)
- [[ETL_vs_ELT]] : 요즘 왜 ETL 대신 ELT를 많이 쓸까? (Modern Data Stack)
- [[Airflow_Architecture(아키텍처)]] : 웹서버, 스케줄러, 워커, 메타DB의 4박자 관계
- [[DAG_Concept]] : DAG가 도대체 뭔가요? (방향성 비순환 그래프)
- [[Airflow_Providers]] : 외부 서비스(AWS, DB, Slack)와 연동하기 위한 확장팩
- [[Execution_Date_Confusion]] :  가장 헷갈리는 '실행 기준일' 개념 잡기 (오늘 돌지만 어제 날짜로 찍히는 이유)

## 1. DAG 작성하기 (Building Pipelines).

> "레고 블록 조립하듯 워크플로우 짜기"

- [[Airflow_DAG_Skeleton]] : `with DAG(...)` 기본 골격 템플릿 (필수 설정값 4대장 포함)
- [[Airflow_DAG_Operators]] : PythonOperator (Classic Way - 아직 많이 씀)
- [[DockerOperator_Usage]] : **"내 파이썬 환경 더럽히기 싫어!" 컨테이너로 태스크 격리해서 실행하기**(`DockeOperator`)
- [[Airflow_TaskFlow_API]] : `@task` 데코레이터로 파이썬 함수처럼 짜기 (Modern Way)
- [[Task_Dependencies]] : 순서 정하기 (`>>`, `[]` 리스트 활용)
- [[Dynamic_Tasks]] : `for`문으로 태스크 대량 생산하기 (Classic Pattern)
- [[Dynamic_Task_Mapping]] : `expand` 함수로 우아하게 매핑하기 (Modern Pattern)
- [[Airflow_Scheduling]] : 스케줄링 정복 (CronTime vs Dataset, Catchup 설정)
- [[Airflow_TaskGroup]] : 태스크들을 폴더처럼 묶어서 정리하기 (시각적 깔끔함)

## 2. 데이터 통신 및 보안 (Data Communication & Security)

> "태스크끼리 데이터를 넘겨주고, 비밀번호는 안전하게 관리해요."

- [[Airflow_XComs]] : 작고 가벼운 데이터 토스하기 (메타데이터 활용)
- [[Variables_Connections]] : 비밀번호(API Key, DB접속정보) 안전하게 관리하기
- [[Airflow_Params]] : DAG 실행 시 설정값 주입하기 (Input / Runtime Config)
- [[Airflow_Hooks]] : Operator가 못하는 복잡한 작업을 위해 '직접' 외부 시스템 문 따고 들어가기

## 3. 워크플로우 제어 및 확장 (Workflow Control & Extension)

> "단순 실행을 넘어, 분기하고 기다리고 시간을 조종하는 고수의 기술."

- [[Airflow_CLI]] : 마우스 대신 키보드로 Airflow 조종하기 (고수의 영역)
- [[Catchup_and_Backfill]] : 과거 데이터 재처리하기 (과거 1년 치 한 번에 돌리기)
- [[Branching]] : 조건에 따라 A경로 혹은 B경로로 분기하기 (`BranchPythonOperator`)
- [[Trigger_Rules]] : "앞에 놈 망해도 나는 실행할래" (태스크 시작 조건 변경)
- [[Airflow_Sensors]] : "파일 들어올 때까지 기다려!" (외부 이벤트 감지)
- [[Airflow_REST_API]] : 파이썬/Curl로 Airflow 원격 조종하기 (외부 트리거)

## 🎯 트러블슈팅 & 운영 (Ops & Debugging)

> "만든 파이프라인을 건강하게 돌리고, 아프면 고쳐줘요."

- [[Common_Airflow_Errors]] : "Task exited with return code 1" 등 자주 보는 에러 모음
- [[Airflow_Best_Practices]] : 실무에서 DAG 짤 때 지켜야 할 십계명 (멱등성 등)
- [[Airflow_Flower]] : 워커(Worker) 상태 감시하기 (Celery 모니터링 CCTV)
- [[Airflow_Queues_Scaling]] : 큐 분리와 성능 튜닝 (Parallelism, Concurrency 완벽 이해)

## 인프라와 외부 연동 (Infrastructure)

> "Airflow 혼자서는 반쪽짜리야. 저장소와 친구들을 연결해 보자."

- [[MinIO_Concept]] : AWS S3를 내 컴퓨터에! (로컬 오브젝트 스토리지) 
-  [[MinIO_Docker_Setup]] : Docker Compose에 MinIO 추가하기 (설치 가이드)
- [[Docker_Compose_Architecture_MinlO_Spark]] : 왜 MinIO는 합치고 Spark는 나눌까? (최종 아키텍처) ⭐️


## 대용량 데이터 처리 (Big Data Processing)

> "엑셀로는 안 열리는 기가바이트(GB), 테라바이트(TB) 단위 데이터를 요리해 보자."

- [[Apache_Spark_Concept]] : 하둡보다 100배 빠른 분산 처리 엔진 
- [[Apache_Spark_Setup]] : Docker로 내 컴퓨터에 Spark 클러스터 구축하기 (설치)
- [[MinIO_Integration]] : Spark(연산)와 MinIO(저장)를 S3 프로토콜로 연결하기
- [[Airflow_Spark_Integration]] : Airflow로 Spark 작업 자동화하기 (SparkSubmitOperator) 
- [[Airflow_Spark_MinIO_Pipeline_Build]]: 이번에 겪은 에러 로그와 해결 과정을 담은 **"실전 문제 해결서"** (나중에 에러 나면 보는 곳)