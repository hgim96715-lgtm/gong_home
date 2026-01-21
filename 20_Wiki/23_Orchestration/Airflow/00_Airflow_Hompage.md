
## 개념 한줄 요약

복잡한 데이터 파이프라인(DAG) 관리 기술을 **'개념(What) -> 작성(How) -> 운영(Run) -> 트러블슈팅(Fix)'** 순서로 정리하여, 전체 흐름을 조망하는 **관제 센터(Control Tower) 노트**입니다.

## 0. 시작하기 (Getting Started) 

> 이론 공부 전, 내 컴퓨터에 Airflow를 띄워봅시다.

- [[Airflow_Installation]] :  Docker vs Standalone 설치 가이드 (맥북 유저 필독!)
- [[Airflow_Configuration]] :  초기 설정 꿀팁 (지저분한 예제 DAG 싹 지우기)
-  [[Docker_Host_Access]] : Docker 컨테이너에서 내 컴퓨터(DB) 접속하는 법 (가장 중요!)
- [[Airflow_UI_Usage]] :  로그 확인 & 재시도 버튼 위치 완벽 정리

## 핵심 철학 (Core Concepts)

> Airflow가 돌아가는 원리 이해하기 (면접 단골 질문!)

-  [[Why_Airflow]] :  데이터 파이프라인과 Airflow 사용 이유 (ETL, 오케스트레이션)
- [[ETL_vs_ELT]] : 요즘 왜 ETL 대신 ELT를 많이 쓸까? (Modern Data Stack)
- [[Airflow_Architecture(아키텍처)]] : 웹서버, 스케줄러, 워커, 메타DB의 4박자 관계
-  [[DAG_Concept]] : DAG가 도대체 뭔가요? (방향성 비순환 그래프)
- [[Airflow_Providers]] :  외부 서비스(AWS, DB, Slack)와 연동하기 위한 확장팩
- [[Execution_Date_Confusion]] : 🚨 가장 헷갈리는 '실행 기준일' 개념 잡기 (오늘 돌지만 어제 날짜로 찍히는 이유)


## 1. DAG 작성하기 (Building Pipelines)

> 레고 블록 조립하듯 워크플로우 짜기

- [[Basic_DAG_Skeleton]] : `with DAG(...)` 기본 골격 템플릿
- [[DAG_Operators_Basic]] : BashOperator, PythonOperator (가장 많이 씀)
- [[Task_Dependencies]] : `>>` (비트 시프트) 연산자로 순서 정하기
- [[Scheduling_Cron]] : 크론표현식(`0 0 * * *`)과 `timedelta` 설정

## 2. 데이터 주고받기 (Data Passing)

> Task A의 결과를 Task B로 넘기는 법

- [[XComs]] : 작고 가벼운 데이터 토스하기 (메타데이터 활용)
- [[Variables_Connections]] : 비밀번호(API Key, DB접속정보) 안전하게 관리하기

## 3. 고급 기능 (Advanced) 

> "오, 좀 치는데?" 소리 듣는 기술들

- [[Catchup_and_Backfill]] : 과거 데이터 재처리하기 (과거 1년 치 한 번에 돌리기)
-  [[Branching]] : 조건에 따라 A경로 혹은 B경로로 분기하기 (`BranchPythonOperator`)
-  [[Sensors]] : "파일 들어올 때까지 기다려!" (외부 이벤트 감지)

## 🎯 트러블슈팅 & 운영 (Ops)

- [[Common_Airflow_Errors]] : "Task exited with return code 1" 등 자주 보는 에러 모음
- [[Airflow_Best_Practices]] : 실무에서 DAG 짤 때 지켜야 할 십계명 (멱등성 등)

