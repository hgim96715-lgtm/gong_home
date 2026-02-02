---
aliases:
  - Job Scheduling
  - Spark Scheduler
  - Dynamic Allocation
  - FAIR Scheduler
tags:
  - Spark
related:
  - "[[Spark_Architecture]]"
  - "[[PySpark_Session_Context]]"
  - "[[00_Apache_Spark_HomePage]]"
---
##  개념 한 줄 요약 

**"한정된 클러스터 자원을 '누가 먼저', '얼마나' 쓸 것인가?"**

스파크의 스케줄링은 크게 두 가지 관점으로 나뉩니다
1.  **클러스터 레벨 (Across Applications):** 여러 스파크 앱끼리 자원 땅따먹기.
2.  **앱 내부 레벨 (Within an Application):** 내 앱 안에서 여러 작업(Job)끼리 순서 정하기.

---
## 애플리케이션 간 스케줄링 (Resource Allocation)

클러스터에 여러 사용자가 동시에 스파크 앱을 제출했을 때, 자원을 어떻게 나눌까요?

### ① Static Allocation (고정 할당) - "알박기"

* **개념:** 앱이 시작할 때 자원을 정해주면, 끝날 때까지 그 자원을 독점합니다
* **장점:** 성능이 예측 가능하고 설정이 단순합니다 
* **단점:** 내가 화장실 간 사이(Idle)에도 자원을 붙들고 있어서, 남들이 못 씁니다 (자원 낭비) 

### ② Dynamic Resource Allocation (동적 할당) - "눈치껏 쓰기" 
️
* **개념:** 바쁠 땐 Executor를 늘리고, 한가할 땐 반납합니다
* **작동 방식:**
    * 할 일(Pending Tasks)이 쌓이면? -> Executor 추가 요청
    * 일 다 하고 놀고 있으면(Idle)?-> Executor 제거 후 반납
* **설정 (Configuration):**

```python
    spark.dynamicAllocation.enabled = true
    spark.dynamicAllocation.minExecutors = 2  # 최소 유지
    spark.dynamicAllocation.maxExecutors = 50 # 최대 확장
    spark.dynamicAllocation.executorIdleTimeout = 60s # 1분 놀면 반납
```

---
## 애플리케이션 내부 스케줄링 (Scheduling Policy)

하나의 `SparkContext` 안에서 여러 쓰레드가 동시에 작업을 던진다면(Action) 순서는 어떻게 될까요? 

### ① FIFO (First In First Out) - "선착순" (기본값)

* **개념:** 먼저 들어온 Job이 자원을 다 쓸 때까지 뒷사람은 기다립니다
* **특징:** 만약 첫 번째 Job이 엄청 크면, 뒤에 있는 작은 Job들은 하염없이 기다려야 합니다

### ② FAIR (Fair Scheduling) - "공평하게"

* **개념:** 여러 Job이 자원을 **Round Robin** 방식으로 나눠 씁니다.
* **특징:**
    * 짧은 작업이 긴 작업 뒤에 있어도 빨리 시작할 수 있습니다.
    * **다중 사용자 환경(예: JupyterHub 공용 노트북)** 에서 필수적입니다.
* **설정:**

```python
    # 1. 스케줄러 모드 변경
    spark.conf.set("spark.scheduler.mode", "FAIR")
    
    # 2. 풀(Pool) 지정 (선택 사항)
    sc.setLocalProperty("spark.scheduler.pool", "production")
```

---
## 비교 요약표 

| 구분              | **Static / FIFO (보수적)**              | **Dynamic / FAIR (유동적)**                       |
| :-------------- | :----------------------------------- | :--------------------------------------------- |
| **Across Apps** | **Static:** 자원 고정. 안정적이지만 낭비 심함.     | **Dynamic:** 자원 유동적. 효율적이지만 초기 지연(Latency) 있음. |
| **Within App**  | **FIFO:** 순서대로 처리. 단순 배치(Batch)에 적합. | **FAIR:** 자원 공유. 대화형(Interactive) 분석에 적합.      |

---
### Tip

"회사 클러스터는 보통 **Dynamic Allocation**이 켜져 있어. 그래서 네가 코드를 멍하니 켜두기만 해도 자원이 자동으로 반납되니까 욕을 덜 먹는 거야.
하지만 **FAIR 스케줄러**는 기본적으로 꺼져 있으니까, 만약 팀원들이랑 하나의 Spark Session을 공유해서 쓴다면 꼭 켜야 해. 안 그러면 한 명이 무거운 쿼리 돌리면 나머지는 다 멈춘다!"