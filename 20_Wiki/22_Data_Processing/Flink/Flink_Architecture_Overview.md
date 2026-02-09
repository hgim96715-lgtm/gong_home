---
aliases:
  - Flink 아키텍처
  - JobManager
  - TaskManager
  - Task Slot
tags:
  - PyFlink
related:
  - "[[Flink_vs_Spark]]"
  - "[[Flink_Introduction]]"
  - "[[00_Apache Flink_HomePage]]"
  - "[[01_Apache Flink_Flow]]"
---
## 개요 (Overview)

Flink 클러스터는 전형적인 **Master-Worker** 아키텍처를 따릅니다.
**JobManager(마스터)** 가 전체 작업을 지휘하고, **TaskManager(워커)** 가 실제 데이터를 처리합니다. 

---
## 주요 컴포넌트 (Components)

### 👑 JobManager (Master Node)
> **"오케스트라의 지휘자"** 

전체 애플리케이션의 실행을 조율(Coordinate)합니다.
* **주요 역할:**
    * **스케줄링 (Scheduling):** 어떤 태스크를 어디서 실행할지 결정. 
    * **체크포인트 관리 (Checkpoint Coordination):** 장애 복구를 위한 스냅샷 저장 명령을 내림. 
    * **리소스 관리 (Resource Mgmt):** 슬롯(Slot)을 할당하고 관리. 
* **세부 구성요소:**
    * **Resource Manager:** K8s, YARN 등과 통신하며 자원(컨테이너)을 따옴. 
    * **Dispatcher:** 클라이언트로부터 Job을 접수하는 REST 인터페이스 제공. 
    * **Job Master:** 개별 JobGraph(실행 계획)의 실행을 전담 관리. 

### 👷 TaskManager (Worker Node)

> **"실제 일하는 작업자"** 

JVM 위에서 동작하며, 실제 데이터 처리 연산(Operator)을 수행합니다.
* **주요 역할:**
    * **태스크 실행:** JobManager가 시키는 대로 코드를 실행. 
    * **데이터 교환:** 다른 TaskManager와 데이터 스트림을 주고받음. 
* **구조:** 최소 1개 이상의 **Task Slot**을 가집니다. 

---
##  핵심 개념: Task Slot & Resources

Flink의 자원 관리에서 가장 중요한 개념입니다.

### 태스크 슬롯 (Task Slot)

* **정의:** TaskManager의 자원(메모리)을 쪼개놓은 **최소 실행 단위(Logical Unit)** 입니다. 
* **특징:**
    * 하나의 슬롯에는 하나의 스레드(Thread)가 할당되어 파이프라인을 실행합니다. 
    * CPU는 격리되지 않지만, **메모리는 격리**됩니다.
* **설정 방법:**
    * `parallelism`(병렬성)과 `taskmanager.numberOfTaskSlots` 설정에 따라 필요한 TaskManager의 개수가 결정됩니다. 
    * 예: 병렬성이 16이고, TM당 슬롯이 4개라면 -> 총 4개의 TaskManager가 필요함. 

###  슬롯 공유 (Slot Sharing)

* Flink는 효율성을 위해 **서로 다른 작업(Subtask)** 들이 하나의 슬롯을 공유할 수 있게 합니다. 
* **장점:** `Source`(데이터 읽기)처럼 가벼운 작업과 `Window`(집계)처럼 무거운 작업을 한 슬롯에 넣어 자원을 골고루 쓰게 합니다.

---
## 실행 모델: Operator Chain

> **"성능 최적화의 핵심"**

Flink는 여러 개의 연산자(Operator)를 하나로 묶어서(Chaining) 실행합니다. 

* **Operator Chain:**
    * 예: `Source` -> `Map` -> `Filter` 과정을 매번 스레드를 바꾸지 않고, **하나의 스레드 안에서 함수 호출하듯이** 연이어 실행합니다. 
    * **이유:** 스레드 간 전환(Context Switching) 비용과 데이터 직렬화/역직렬화 비용을 줄여서 **성능을 극대화**하기 위함입니다.
* **Task vs Subtask:**
    * **Task:** Operator Chain으로 묶인 한 덩어리.
    * **Subtask:** 그 Task가 병렬성(Parallelism)에 의해 여러 개로 복제된 실행 단위 (실제 스레드). 

![[스크린샷 2026-02-05 오후 5.32.37.png]]


---
## 전체 동작 흐름 (Dataflow)

1. **Client:** 코드를 짜서 JobManager에게 제출(Submit). 
2. **JobManager:** 코드를 분석해 **실행 계획(Dataflow Graph)** 을 만들고 스케줄링. 
3. **ResourceManager:** 필요한 만큼의 TaskManager(컨테이너)를 확보. 
4. **TaskManager:** 할당받은 **Slot**에 **Task**를 띄워서 데이터를 처리. 
5. **Heartbeats:** 작업자들은 주기적으로 마스터에게 "나 살아있어"라고 보고. 

![[스크린샷 2026-02-05 오후 5.32.15.png]]