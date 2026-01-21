---
aliases:
  - Airflow 아키텍처
  - Airflow 구성요소
  - Scheduler
  - Worker
  - Executor
tags:
  - Airflow
  - 면접질문
related:
  - "[[00_Airflow_Hompage]]"
---
## 개념 한 줄 요약

Airflow는 혼자서 일하는 게 아니라, **지시하는 놈(Scheduler), 기록하는 놈(Meta DB), 보여주는 놈(Web Server), 실제로 일하는 놈(Worker)** 이 철저하게 분업화된 **분산 처리 시스템**입니다.

---
## 왜 필요한가 (Why)

**문제 상황:**
만약 하나의 컴퓨터 프로그램이 "스케줄링도 하고, 화면도 보여주고, 무거운 데이터 처리도 직접 한다"고 상상해 봅시다. 
사용자가 웹 화면을 새로고침하다가 프로그램이 멈추면, 밤새 돌아야 할 데이터 파이프라인도 같이 죽어버립니다. (단일 실패 지점)

**해결책:**
역할을 쪼갭니다(Decoupling). 
* 웹 서버가 터져도 스케줄러는 계속 돕니다.
* 워커 하나가 죽어도 다른 워커가 일을 대신 받습니다.
* 이렇게 해서 시스템의 **안정성(Stability)** 과 **확장성(Scalability)** 을 확보합니다.

---
## 실무 맥락에서의 사용 이유

서버가 느려지거나 에러가 났을 때, 엔지니어는 **"어디가 아픈지"** 정확히 찔러야 합니다.
* "DAG가 안 보여요" -> **Web Server**나 **DAG Directory** 확인
* "시간이 됐는데 작업이 시작을 안 해요" -> **Scheduler**가 죽었거나 멈춤
* "계속 Running 상태에서 안 넘어가요" -> **Worker**나 **Executor** 문제
이 구조를 알아야 트러블슈팅이 가능합니다.

---
## 핵심 구성요소 (Key Components)

### 1. The Brain & Heart 

* **Scheduler (스케줄러):**
    * 가장 중요합니다. "언제(When) 어떤 작업(Task)을 실행해야 하는지" 결정합니다.
    * DAG 파일을 읽어서 파싱하고, 실행할 때가 되면 일감을 Executor에게 넘깁니다.
* **Metadata Database (메타 데이터베이스):**
    * Airflow의 '기억'입니다. DAG가 성공했는지 실패했는지, 유저 정보는 무엇인지 등 모든 **상태(State)** 를 저장합니다.
    * 보통 PostgreSQL이나 MySQL을 사용합니다.

### 2. The Muscle 

* **Executor (실행자):**
    * **"어떻게(How) 실행할 것인가"** 를 결정하는 매커니즘입니다.
    * `LocalExecutor` (한 컴퓨터에서 실행), `CeleryExecutor` (여러 컴퓨터에 분산 실행), `KubernetesExecutor` (파드 띄워서 실행) 등이 있습니다.
* **Worker (워커):**
    * 실제로 땀 흘려 일하는 프로세스입니다.
    * Executor가 큐(Queue)에 던져준 일감을 가져와서 Python 코드를 실행하거나 쿼리를 날립니다.

### 3. The Interface 

* **Web Server (웹 서버):**
    * 우리가 보는 파란색 Airflow 웹 화면입니다. 사용자가 로그를 보거나 수동으로 DAG를 켜는(Trigger) 역할을 합니다.
    * 얘는 실제로 작업을 실행하지 않습니다. 그냥 DB에 있는 걸 보여줄 뿐입니다.
* **DAG Directory:**
    * 우리가 짠 Python 파일(`my_dag.py`)들이 저장된 폴더입니다. 스케줄러가 여기를 주기적으로 읽습니다.

---
## 아키텍처 흐름도 (작동 원리)

1.  **Scheduler**가 DAG 폴더를 읽어서 "아, 9시에 돌 작업이 있네?" 하고 감지합니다.
2.  **Scheduler**가 작업 상태를 'Scheduled'로 바꾸고 **Executor**에게 전달합니다.
3.  **Executor**는 작업을 **Queue**에 넣습니다. (RabbitMQ, Redis 등)
4.  놀고 있던 **Worker**가 Queue에서 작업을 낚아채서 실행합니다.
5.  실행 결과(성공/실패)는 **Metadata DB**에 기록되고, 우리는 **Web Server**를 통해 그걸 봅니다.

---
## 배포 구성: 싱글 노드 vs 멀티 노드

Airflow를 실제로 설치할 때, 위에서 배운 컴포넌트들을 **컴퓨터 한 대에 몰아넣느냐(Single)**, **여러 대에 나누느냐(Multi)** 에 따라 구조가 달라집니다.

### 1. 싱글 노드 아키텍처 (Single Node)

> "한 지붕 한 가족" (모든 컴포넌트가 서버 1대에 존재)

* **구조:** Web UI, Scheduler, Executor, Queue, Metadata DB가 전부 하나의 머신(Node) 안에 설치됩니다.
* **용도:** POC(개념 증명), 학습용, 혹은 매우 가벼운 개인 프로젝트용.
* **특징:**
    * 설치가 매우 쉽습니다. (`pip install` 하고 실행하면 끝)
    * **Executor:** 주로 `LocalExecutor`를 사용합니다.
    * **단점:** 서버가 죽으면 Airflow 전체가 죽습니다. 작업량이 많아지면 버티지 못합니다.

### 2. 멀티 노드 아키텍처 (Multi Nodes)

> "대기업의 분업화" (역할별로 서버를 쪼개서 운영)

* **구조:**
    * **Node 1 (마스터):** Web UI, Scheduler, Executor가 위치하여 지휘를 내립니다.
    * **Node 2 (저장소):** Metadata DB만 따로 두어 안정성을 확보합니다.
    * **Node 3 (큐):** Redis나 RabbitMQ 같은 Queue 시스템을 독립시킵니다.
    * **Node N (워커):** 실제 일을 하는 Worker들을 여러 대(Scale-out)로 늘립니다.
* **용도:** 실제 회사 서비스(Production) 환경.
* **특징:**
    * **확장성:** 일이 많아지면 **Worker 노드**만 계속 추가하면 됩니다.
    * **Executor:** `CeleryExecutor`나 `KubernetesExecutor`를 사용하여 네트워크를 통해 일을 뿌려줍니다.
    * **안정성:** Worker 하나가 고장 나도, 다른 Worker가 일을 대신 처리하므로 서비스가 멈추지 않습니다.
---
## 초보자가 자주 착각하는 포인트

1.  **"Executor가 작업을 실행한다?"**
    * 반은 맞고 반은 틀립니다. Executor는 **"작업을 배달하는 관리자"** 에 가깝고, 실제 코드를 돌리는 건 **Worker**입니다. (LocalExecutor인 경우엔 한 프로세스 안에 같이 있긴 합니다.)
2.  **"Web Server가 죽으면 스케줄도 멈추나요?"**
    * 아닙니다! 웹 서버는 단지 모니터(화면)일 뿐입니다. 화면이 꺼져도 본체(Scheduler)는 살아서 계속 작업을 돌립니다.
3.  **Task vs Task Instance**
    * **Task:** 코드에 적혀있는 작업 그 자체 (클래스). "매일 리포트 보내기"라는 정의.
    * **Task Instance:** 그 작업이 특정 날짜에 실행된 실체 (인스턴스). "2026년 1월 21일자 리포트 보내기 작업".
    * 로그를 볼 때는 항상 **Task Instance**의 로그를 보는 것입니다.