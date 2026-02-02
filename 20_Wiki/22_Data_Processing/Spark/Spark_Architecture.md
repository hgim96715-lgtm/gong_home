---
aliases:
  - Driver
  - Executor
  - ClusterManager
  - 스파크구조
  - MasterNode
  - WorkerNode
tags:
  - Spark
related:
  - "[[Spark_Installation_Local]]"
  - "[[Spark_Concept_Evolution]]"
  - "[[Process_vs_Thread]]"
  - "[[Spark_Submit_Command]]"
---
##  개념 한 줄 요약

**"스파크는 '시키는 놈(Driver)', '중개하는 놈(Cluster Manager)', '일하는 놈(Executor)'의 3박자로 돌아가는 분산 공장이다."**

* **Driver:** 작업 반장. (계획 수립, 지시)
* **Cluster Manager:** 인력 사무소. (자원 할당, 채용)
* **Executor:** 현장 일용직. (실제 작업 수행, 캐싱)

>"면접관이 '스파크 구조 설명해보세요' 하면 딱 세 단어만 기억해.
> **Driver(반장), Manager(인사팀), Executor(일꾼)**. 
> 그리고 **'Driver는 뇌를 쓰고, Executor는 힘을 쓴다'**

---

## 핵심 3요소 (The Holy Trinity)

스파크 클러스터는 크게 **Master Node(관리자)** 와 **Worker Node(일꾼)** 라는 물리적 서버들로 구성되며
그 위에서 다음 3가지 소프트웨어 프로세스가 돌아갑니다.

### ① Driver Program (The Brain )

* **위치:** Master Node에서 주로 실행됩니다 (모드에 따라 다름). 
* **역할:**
    * 우리가 짠 코드(`main()`)를 실행하는 주체입니다.
    * **SparkContext**를 생성하여 클러스터와 연결합니다. 
    * 거대한 작업을 작은 **Task**로 쪼개서 일꾼들에게 나눠줍니다. 
* **비유:** "건설 현장의 **현장 소장**." (설계도를 들고 지시함)

### ② Cluster Manager (The HR Manager )

* **위치:** Master Node 또는 별도의 자원 관리 서버.
* **역할:**
    * 전체 서버(Cluster)의 CPU와 메모리 상태를 파악하고 있습니다.
    * Driver가 "일꾼 3명만 보내줘!" 하면 놀고 있는 Worker Node에서 자원을 떼어줍니다.
* **종류:**
    * **YARN:** 하둡 생태계의 표준 (가장 많이 씀). 
    * **Kubernetes (K8s):** 요즘 뜨는 컨테이너 기반 관리자.
    * **Standalone:** 스파크 자체 내장 매니저 (간단한 테스트용). 

### ③ Executor (The Muscle )

* **위치:** Worker Node 안에서 실행되는 **프로세스(Process)** 입니다. 
* **역할:**
    * river가 시킨 **Task**를 실제로 계산합니다
    * 처리 중인 데이터를 **Cache(메모리)**에 저장해둡니다. (속도의 핵심!) 
* **비유:** "벽돌 나르는 **인부**." (시키지 않은 일은 절대 안 함)

---

## 작동 흐름 (Workflow)

스파크 작업(Job)을 제출(`spark-submit`)하면 내부적으로 이런 일이 벌어집니다. (YARN 기준) 

1.  **제출 (Submission):** 사용자가 코드를 실행하면 **Driver**가 켜집니다.
2.  **자원 요청:** Driver가 **Cluster Manager**에게 "나 이 작업 하려면 Executor 5개 필요해"라고 요청합니다. 
3.  **자원 할당:** Cluster Manager가 각 **Worker Node**에 지시하여 **Executor(Container)** 를 띄웁니다. 
4.  **작업 수행:** Driver가 Executor들에게 **Task(할 일)** 와 **Jar(코드)** 를 전송합니다.
5.  **결과 반환:** Executor들이 열심히 계산해서 결과를 Driver에게 보고하거나 파일로 저장합니다.

---

## 심화: PySpark는 어떻게 돌아가나? (Py4J)

우리는 Python을 쓰지만, 스파크는 원래 Scala(Java)로 만들어졌습니다. 
서로 언어가 다른데 어떻게 대화할까요? 

* **Python Wrapper:** 우리가 짠 파이썬 코드는 껍데기일 뿐입니다.
* **Py4J:** 파이썬에서 자바 객체(JVM)를 건드릴 수 있게 해주는 **통역사**입니다. 
* **작동 방식:**
    1.  내가 `df.filter()` (파이썬)를 실행함.
    2.  **Py4J**가 이걸 알아듣고 JVM(자바 가상 머신)에 있는 실제 스파크 함수를 호출함. 
    3. Worker Node에서도 `Python Worker`가 따로 떠서 파이썬 관련 로직(UDF 등)을 처리함. 
---

## 초보자가 자주 하는 실수 (Misconceptions)

### ① "Worker Node랑 Executor는 같은 건가요?"

* **다릅니다!** 
* **Worker Node:** 물리적인 컴퓨터(서버) 1대. (하드웨어)
* **Executor:** 그 컴퓨터 안에서 실행되는 프로그램(프로세스). (소프트웨어) 
* *하나의 Worker Node(32코어) 안에 Executor(4코어짜리) 8개가 들어갈 수도 있습니다.*

### ② "Driver 메모리는 왜 중요한가요?"

* Executor들이 계산한 결과를 `collect()` 같은 함수로 모으면 전부 **Driver 메모리**로 쏠립니다.
* 이때 Driver 메모리가 작으면 **"Out Of Memory (OOM)"** 가 터지면서 프로그램이 죽습니다. (반장 머리가 터짐 )

