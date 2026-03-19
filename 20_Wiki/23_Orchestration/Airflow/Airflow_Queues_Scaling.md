---
aliases:
  - Airflow Queue
  - Parallelism
  - Concurrency
  - Max Active Runs
  - 성능 튜닝
tags:
  - Airflow
  - Ops
  - Performance
  - Scaling
related:
  - "[[Airflow_Flower]]"
  - "[[Airflow_Architecture(아키텍처)]]"
  - "[[CS_Stack_Queue]]"
  - "[[CS_Blocking_vs_NonBlocking]]"
---
## 개념 한 줄 요약

**Airflow Queue**는 작업의 성격(CPU 작업 vs 메모리 작업)에 따라 **"줄 서는 라인"을 분리**하는 기술이고, **Concurrency 설정**들은 각 라인에서 **"한 번에 몇 명까지 통과시킬지" 정하는 속도 제한 표지판**이야.

---
### 💡 용어 정리: IO_Intensive vs CPU_Intensive

큐 이름을 왜 이렇게 지었을까요? 
작업의 성격에 따라 필요한 **컴퓨터 자원**이 다르기 때문입니다.

**1. IO_Intensive (입출력 집중형) = "기다리는 작업"**
* **의미:** CPU(머리)는 거의 안 쓰고, **Input/Output(네트워크, 디스크)** 응답을 기다리는 데 시간을 다 쓰는 작업들입니다.
* **예시:**
    * DB에 쿼리 날리고 결과 기다리기 (`SELECT * FROM ...`)
    * 외부 API 호출하고 응답 기다리기 (`requests.get`)
    * 파일 다운로드/업로드
    * `sleep 20` (그냥 멍 때리고 기다리기)
* **전략:** 머리를 안 쓰니까, **한 번에 수백 개를 실행(Concurrency 높임)** 해도 컴퓨터가 안 느려집니다.
* **비유:** **"배달 음식 주문해놓고 기다리기"** (동시에 10군데 주문해도 나는 안 힘듦).

**2. CPU_Intensive (계산 집중형) = "머리 쓰는 작업"**
* **의미:** 외부 대기 없이, **CPU(머리)** 가 쉴 새 없이 돌아가며 계산해야 하는 작업들입니다.
* **예시:**
    * 동영상 인코딩/압축
    * 복잡한 암호화/복호화
    * 머신러닝 모델 학습 (Training)
    * Pandas로 거대한 데이터프레임 연산
* **전략:** 머리가 터지기 일보 직전이라, **코어 수(Core Count) 만큼만 실행**해야 효율적입니다. 무리하게 많이 시키면 컴퓨터가 멈춥니다.
* **비유:** **"고난도 수학 문제 풀기"** (동시에 10문제를 풀 수는 없음, 하나씩 풀어야 함).

---
## 왜 필요한가 (Why)

 **문제점 (Bottleneck):** 
 * 무거운 머신러닝 학습 작업이랑 가벼운 이메일 전송 작업이 같은 줄(Queue)에 서 있으면, 무거운 작업 때문에 가벼운 작업도 하루 종일 기다려야 해.
* 제한 없이 작업을 막 실행하면 서버 CPU가 터져서(OOM) Airflow 전체가 멈춰버려.

 **해결책:**
-  **Queue 분리:** "무거운 놈들은 A라인, 가벼운 놈들은 B라인으로 가!"
* **Concurrency 제한:** "우리 서버는 한 번에 32개까지만 감당 가능하니까 그 이상은 대기시켜."

---
## Practical Context (실무 활용)

현업에서는 이렇게 씁니다.

 **Queue 활용:** 
 - `cpu_intensive` 큐(고성능 서버)와 `light_task` 큐(저사양 서버)를 나눠서 비용을 절감함.
 
**튜닝 시점:** 
- 태스크들이 실행되지 않고 `Scheduled`나 `Queued` 상태에서 하염없이 멈춰있을 때 이 숫자들을 조절함.
---
##  Code Core Points (Queue 지정하기)

### A. 개념: "편지 봉투에 수신자 적기"

* **DAG의 `queue="cpu_intensive"`:**
    * 작업을 보낼 때 봉투 겉면에 **"이건 CPU 팀만 뜯으시오"** 라고 수신자를 적는 행위야.
    * 이걸 안 적으면 기본값인 `"default"`라고 적혀서 발송돼.

* **Docker의 `command: celery worker -q cpu_intensive`:**
    * 워커(직원)를 출근시킬 때 **"너는 'CPU 팀' 우편함만 확인해!"** 라고 업무 지시를 내리는 거야.
    * `-q` 옵션이 없으면 `"default"` 우편함만 뒤적거려.

### B. 실전 코드 매핑 (User Example)

네가 작성한 코드가 실제로 어떻게 돌아가는지 그림으로 그려보면 이렇습니다.

**1. DAG (작업 요청서)**

```python
task_producer1 = BashOperator(
    task_id="sleep_1",
    queue="cpu_intensive",  # 👈 "이건 cpu_intensive 큐로 보내세요"
    ...
)

task_producer2=BashOperator(
	task_id="sleep_2",
	queue="io_intensive",
	bash_command='sleep 20'
)
```

**2. Docker Compose (작업 처리반)**

- **airflow-worker2:** `command: celery worker -q cpu_intensive`
	- `task_producer1`을 가져감.
	- `task_producer2`("io_intensive")는 쳐다도 안 봄.

- **airflow-worker3:** `command: celery worker -q io_intensive`
	-  `task_producer2`를 가져감.

### C. Flower 모니터링 해석

실습 후 **Flower** 화면에 들어갔을 때 아래와 같은 화면이 떴다면 **성공**이야.

- **Name: `io_intensive`**: "현재 `io_intensive`라는 이름의 큐가 생성되어 있다."
- **Active Consumers**: 여기에 숫자가 `1`이라고 떠야 해. 
	- 즉, **"이 큐를 감시하고 있는 워커(airflow-worker3)가 살아있다"** 는 증거야.
	- **만약 이 줄이 없다면?**: `airflow-worker3`가 죽었거나, `-q` 옵션 오타로 인해 엉뚱한 큐를 보고 있다는 뜻.

---
## Detailed Analysis (성능 튜닝 4대장)

### 1) `core.parallelism = 32` (전체 총량)

- **의미:** Airflow 전체에서 **동시에 돌아갈 수 있는 태스크의 최대 개수**.
- **해석:** DAG가 100개든 1000개든 상관없이, 지금 이 순간 Airflow 전체가 처리하는 태스크는 **절대 32개를 넘을 수 없어.** 가장 강력한 제약이야.

### 2) `core.max_active_tasks_per_dag = 16` (DAG당 제한)

- **의미:** **하나의 DAG 안에서** 동시에 실행될 수 있는 태스크 수.
- **해석:** 전체 용량(32)이 남아있어도, 특정 DAG 하나가 혼자서 32개를 다 쓰면 안 되잖아? 그래서 "너는 최대 16개까지만 써"라고 제한을 거는 거야.

### 3) `core.max_active_runs_per_dag = 16` (DAG 실행 제한)

- **의미:** 하나의 DAG가 **동시에 배포(Run)될 수 있는 횟수**.
- **해석:** 만약 `Catchup=True`라서 과거 1년 치를 돌려야 한다면? 이 설정이 `16`이면 **16일 치 데이터만 동시에 돌고**, 나머지는 대기해. (과부하 방지)

### 4) `celery.worker_concurrency = 16` (워커 체력)

- **의미:** **워커(컴퓨터) 한 대가** 동시에 처리할 수 있는 태스크 수.
- **해석:** 보통 CPU 코어 수와 비슷하게 맞춰. 워커가 2대 있고 각각 16이라면, 총 32개의 일을 할 수 있겠지? 이게 `parallelism` 설정과 비슷하거나 커야 효율적이야.

---
## 초보자가 자주 착각하는 포인트

1. "Queue 이름만 적으면 알아서 가나요?"
	- 아니! 반드시 **그 Queue 이름을 감시하는 Worker를 따로 띄워야 해.** 
	- (`celery worker -q my_queue` 처럼). 
	- 받는 놈이 없으면 태스크는 영원히 대기해.

2. "숫자를 무조건 크게 하면 빨라지나요?"
	- 절대 아님. 
	- CPU 코어는 4개뿐인데 `concurrency`를 100으로 잡으면, 컴퓨터가 문맥 교환(Context Switching) 하느라 오히려 느려지고 멈출 수 있어.

3. LocalExecutor에서도 되나요?
	- `Queue` 기능은 기본적으로 **CeleryExecutor**나 **KubernetesExecutor** 같은 분산 환경에서만 작동해.