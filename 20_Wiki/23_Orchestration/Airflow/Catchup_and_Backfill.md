---
aliases:
  - Backfill
  - Catchup
  - 백필
  - 과거 데이터 처리
  - 재처리
tags:
  - Airflow
  - Operations
  - Scheduling
related:
  - "[[Airflow_CLI]]"
  - "[[Airflow_DAG_Skeleton]]"
---
## 개념 한 줄 요약

**Backfill(백필)** 은 Airflow의 **"타임머신"** 기능이야.
현재 시점(`Today`)에서 과거 특정 기간(`2024-11-01` ~ `2024-11-04`)의 DAG를 강제로 실행시켜서 **비어있는 데이터를 채우거나(Fill in), 잘못된 데이터를 덮어쓰는 작업**을 말해.

> **⭐️ 핵심 작동 원리 (Mechanism)** 
>  **Backfill의 역할:** 과거의 **DAG Run(작업 지시서)을 대량으로 생성(Bulk Create)** 하는 것까지야. 
>   **Scheduler의 역할:** 쌓여있는 지시서(DAG Run)를 보고 실제로 태스크를 **실행(Execute)** 하는 건 스케줄러가 담당해. 
>   👉 **결론:** 백필 명령은 "주문서를 왕창 넣는 행위"일 뿐이고, 요리는 스케줄러가 알아서 순서대로 처리한다!


---
## 왜 필요한가 (Why)

**Catchup과의 차이:**
- **Catchup=True:** DAG를 켜자마자 과거부터 밀린 숙제를 **자동으로** 미친 듯이 해치움. (위험할 수 있음 ⚠️)
- **Backfill:** 내가 원할 때 **수동으로** 특정 기간만 콕 집어서 실행함. (안전함 ✅)

**사용 시나리오:**
- **시스템 장애 복구:** 서버가 죽어서 3일 치 데이터가 빵꾸났을 때.
- **로직 수정 후 재처리:** "아, 지난달 매출 계산 공식이 틀렸네?" -> 코드 수정 후 지난달 것만 다시 돌리기.
- **초기 데이터 적재:** `catchup=False`로 두고, 오픈 전 1년 치 데이터는 Backfill로 한 번에 넣을 때.

---
## Practical Context (실전 가이드)

> **💡 사전 준비 (Alias 설정)**
> * **[Docker 원본]:** `docker exec -it apache-airflow-airflow-scheduler-1 airflow ...`
> * **[Alias 사용]:** `air ...` ([[Airflow_CLI#**2. 매번 `docker exec...` 치기 귀찮아! (Alias 설정)**|Alias사용하기]]에서 설정한 별칭)

### A. 백필 생성 (Create)

Airflow 2.10+ / 3.0부터는 백필이 **관리형 객체**로 승격됐어.
터미널을 꺼도 서버가 알아서 돌려줘! 
**주의:** 생성할 때는 반드시 **`--dag-id` 옵션**을 명시적으로 적어줘야 해!

```bash
# 문법: air backfill create --dag-id <DAG_ID> --from-date <시작> --to-date <끝> 
air backfill create --dag-id backfilling_sample \
 --from-date 2026-01-02 \ 
 --to-date 2026-01-05
```

**주요 옵션 (Argument):**
- `--dag-id` : 백필을 수행할 **DAG ID**
- `--from-date` : 백필 시작 날짜 (inclusive)
- `--to-date` : 백필 종료 날짜 (inclusive)

**Tip:** 날짜 옵션이 버전마다 `-s`/`-e` 일 수도 있고 `--from-date` 일 수도 있어. 
헷갈리면 무조건 **`air backfill create --help`** 를 쳐보는 게 정답이야.

### B. 결과 확인 (Check Runs)

"백필 Job"을 찾는 게 아니라, **"생성된 DAG Run들"** 을 확인해야 해. 
(조회할 때는 DAG ID를 그냥 맨 뒤에 적어도 되는 경우가 많아.)

```bash
#  docker exec -it apache-airflow-airflow-scheduler-1 airflow dags list-runs backfilling_sample
# 문법: air dags list-runs <DAG_ID>
air dags list-runs backfilling_sample
```

#### 출력 결과 해석 (Decoding Output)

터미널 로그는 **최신 날짜가 맨 위(Top)** 에 온다는 걸 기억해!

|**컬럼**|**설명**|**포인트 👁️**|
|---|---|---|
|**run_id**|주민번호|**앞머리(접두어)** 를 보면 누가 실행했는지 알 수 있어! (아래 참고)|
|**logical_date**|**데이터 날짜**|네가 입력한 `--from-date` 날짜가 여기 찍혀.|
|**state**|상태|`success` (성공) / `running` (도는 중) / `failed` (실패)|

##### 💡 Run ID 3대장

접두어를 보면 "누가 이 DAG를 실행시켰는지" 범인을 알 수 있어.

- **`backfill__...` (백필):**
    - CLI로 `backfill create` 명령을 내려서 강제로 돌린 놈. ****

- **`manual__...` (수동):**
    - 웹 UI에서 사용자가 **재생(▶️) 버튼**을 눌러서 실행한 놈.

- **`scheduled__...` (자동):**
	-  Airflow가 **스케줄(`@daily` 등)** 에 맞춰서 제시간에 알아서 돌린 놈.
	- 새벽에 몰래 돌고 성공해 있는 기특한 녀석.

**[실전 예시 로그]**

```text
dag_id               | run_id                          | state    | logical_date
---------------------+----------------------------------+----------+---------------------
backfilling_sample   | manual__2026-01-23...           | failed   | 2026-01-23 ...   (내가 버튼 누름 💥)
backfilling_sample   | scheduled__2026-01-23...        | success  | 2026-01-23 ...   (자동으로 돔 🤖)
backfilling_sample   | backfill__2024-01-03...         | success  | 2024-01-03 ...   (백필 명령 내림 🛠️)
```


### C. 백필 결과 삭제 (Clean Up)

CLI로 날짜 범위(`--start-date` ~ `--end-date`)를 지정해서 지우는 건 **절대 비추천**이야. (정상 스케줄까지 날아감 😱)

- **강추:** Web UI (Grid View -> DAG Runs -> Backfill 필터 -> 삭제)
- **CLI:** ID로 콕 집어 삭제

```bash
docker exec -it apache-airflow-airflow-scheduler-1 airflow dags delete-dag-run \
    backfilling_sample --run-id backfill__2026-01-02...
```

---
## Code Core Points (실습 예제)

```python
import pendulum
from airflow.decorators import dag, task

@dag(
    dag_id='backfilling_sample',
    start_date=pendulum.datetime(2026, 1, 2, tz="America/New_York"),
    catchup=False, 
    schedule='@daily'
)
def backfilling_sample():
    @task(retries=3)
    def print_context():
        print("Hello Backfill!")
        # 🚨 테스트를 위해 고의로 에러 발생!
        raise Exception("Backfill Test Error")
    print_context()

backfilling_sample()
```

---
## Detailed Analysis (재처리 워크플로우)

**상황:** 위 코드를 배포했고, 백필을 돌렸는데 에러가 나서 `failed` 상태라고 가정하자. 
코드를 수정하고 **"실패한 태스크만"** 다시 돌리고 싶다면?

**1. 백필 요청 (Create)** 
일단 없는 DAG Run을 만들어. (Docker 명령어는 위 A섹션 참고)

```bash
air backfill create --dag-id backfilling_sample ...
```

**2. 실패 확인 (List)**

```bash
air dags list-runs backfilling_sample
# 결과: run_id가 backfill__... 이고 state가 failed 인 것 확인
```

**3. 실패한 태스크만 재실행 (Tasks Clear)**

- **핵심** `backfill create`를 또 하는 게 아니야! **`tasks clear`** 명령어로 "상태를 초기화"해주는 거야.

```bash
air tasks clear backfilling_sample \
  --only-failed \
  --start-date 2026-01-02 \
  --end-date 2026-01-05
  
# docker exec -it apache-airflow-airflow-scheduler-1 airflow tasks clear backfilling_sample \
#   --only-failed \
#  --start-date 2026-01-02 \
#  --end-date 2026-01-05
```

**원리:** Task 상태를 `Failed` ➔ `None`으로 바꾸면, 스케줄러가 "어? 이거 해야겠네?" 하고 다시 실행함.

---
## Run 단위 vs Task 단위 (헷갈리지 말자!)

|**목적**|**상황**|**명령어 (Action)**|
|---|---|---|
|**백필 요청 자체**|"과거 DAG Run이 아예 없어. 만들어줘."|`backfill create --dag-id ...`|
|**실패 태스크 재실행**|"Run은 있는데 실패했어. 걔만 다시 돌려."|`tasks clear --only-failed ...` ✅|
|**특정 DAG Run 삭제**|"잘못 만들었어. 기록 삭제할래."|`dags delete-dag-run ...`|

---
## 내가 헷갈리는 것 Retries vs Backfill

> Retries는 [[Airflow_DAG_Skeleton#④ `default_args` 귀찮은 설정 상속하기| Retries에 대해 ]] 참조 

두 가지는 **"실패를 처리한다"** 는 점에서는 비슷해 보이지만, **스케일(범위)과 시점**이 완전히 다릅니다.

- **Retries:** "어? 잠깐 삐끗했네? **지금 당장** 다시 해봐." (자동 반사)
- **Backfill:** "아, 옛날 데이터가 통째로 없거나 틀렸네? **과거로 가서** 다시 만들어." (시간 여행)

### 비유로 이해하기 (게임 vs 현실)

| **구분**        | **비유 (상황)**                                                                               | **핵심**                                |
| ------------- | ----------------------------------------------------------------------------------------- | ------------------------------------- |
| **Retries=3** | **게임하다 죽었을 때 (부활)**<br><br><br>보스 잡다가 죽음 👉 "Continue? (3/3)" 👉 제자리에서 즉시 부활해서 다시 싸움.     | **자동 & 즉시**<br><br>  <br>(일시적 오류 해결용) |
| **Backfill**  | **챕터 다시 하기 (회상)**<br><br><br>게임을 다 깼는데, 1스테이지 아이템을 안 먹었음 👉 "챕터 선택" 메뉴로 가서 1탄을 처음부터 다시 깸. | **수동 & 과거**<br><br><br>(데이터 복구/재적재용)  |

### 기술적 차이점 (Technical Deep Dive)

#### A. Retries (재시도) - "끈기 있는 좀비" 🧟‍♂️

코드에 적은 `@task(retries=3)`은 **DAG가 실행되는 도중(Runtime)** 에 일어나는 일이야.

- **언제:** 태스크가 실패(`Failed`)하자마자.
- **누가:** 워커(Worker)가 알아서. (사람이 개입 X)
- **왜:**
    - "API가 1초 동안 끊겼어."
    - "DB 커넥션이 잠깐 타임아웃 났어."
    - 👉 **"잠깐 기다렸다 다시 하면 될 것 같은데?"** 싶을 때 씀.
- **결과:** 3번 다 실패하면 그제야 비로소 **`Failed`** 상태로 죽어버림. (이때 알람이 옴 🚨)

#### B. Backfill (백필) - "타임머신 주문서" 📜

우리가 CLI로 친 `backfill create`는 **DAG 실행 자체를 생성**하는 일이야.

- **언제:** 데이터가 비었거나 로직이 수정되었을 때 (보통 나중에).
- **누가:** 엔지니어(너)가 수동으로.
- **왜:**
    - "서버가 죽어서 3일 치 데이터가 아예 없어."
    - "매출 계산 공식이 틀려서 지난달 거 싹 다 다시 계산해야 해."
- **결과:** 지정한 기간의 DAG Run들이 생성되고, 스케줄러가 이걸 가져가서 실행함.

### 둘의 관계 (함께 동작한다!)

**Backfill을 돌릴 때도 Retries는 동작해!**

- 네가 터미널에서 **Backfill 명령**(`air backfill create...`)을 내림. (주문서 투입)
- Airflow가 과거 날짜의 DAG를 실행함.
- 돌아가다가 에러가 남! 
- 이때 **Retries=3** 설정이 있으니까, Airflow는 바로 포기하지 않고 **3번까지 혼자서 다시 시도**함.
- 그래도 안 되면 최종 실패(`Failed`).
- 넌 나중에 와서 "어? 백필 돌려놓은 거 실패했네?" 하고 **`tasks clear`** 로 다시 살려냄.

### 요약

|**특징**|**Retries (재시도)**|**Backfill (백필)**|
|---|---|---|
|**성격**|**응급처치 (Band-aid)** 🩹|**수술/재건축 (Reconstruction)** 🏗️|
|**목적**|일시적 네트워크/DB 오류 자동 극복|과거 데이터 누락 채우기, 로직 수정 반영|
|**발동 시점**|에러 난 **직후** (자동)|엔지니어가 **명령을 내릴 때** (수동)|
|**대상**|태스크 (Task) 하나|DAG Run 전체 (기간)|

>`Retries`는 **"새벽 3시에 날 깨우지 않게 해주는 고마운 기능"** 이고,
> `Backfill`은 **"출근해서 사고 친 데이터를 수습하는 기능"** 이야. 
> 둘 다 없으면 안 돼!

---
## 초보자가 자주 착각하는 포인트

1. "Start Date랑 End Date가 헷갈려요."
	- Backfill은 `-s` 이상, `-e` **미만**이 아니라 **이하(포함)** 일 때도 있고 버전마다 미묘하게 다를 때가 있어.
	- 항상 **"작은 범위로 먼저 테스트"** 해보고 돌려!

2. "Backfill 돌리면 기존 스케줄은 멈추나요?"
	- 아니! 오늘 돌아야 할 스케줄은 그대로 돌고, Backfill은 별도의 프로세스로 동시에 돌아가. 
	- (서버 부하 주의 )

3. **"스케줄 없는 DAG는 Backfill도 못 해!" (가장 중요 ⭐️)**
	- `schedule=None` (수동 트리거 전용)으로 설정된 DAG는 백필을 돌릴 수 없어.
	- **이유:** Backfill의 원리가 "과거의 **스케줄(Interval)** 을 강제로 실행하는 것"이기 때문이야. 스케줄이 없으면 "언제 실행됐어야 하는지" 기준이 없어서 에러가 나거나 아무 일도 안 일어나.
	- **해결책:** Backfill을 하고 싶다면 반드시 `@daily`, `@hourly` 같은 **스케줄이 정의되어 있어야 해.**

4. **"백필 옵션에 `--rerun-failed` 없나요?"**
	- 없어졌어. 
	- 생성(`create`)과 수정(`clear`)은 역할이 분리됐어.
	- `tasks clear`를 써야 해.

>실무에서는 **`catchup=True`를 거의 쓰지 않습니다.** (실수로 서버 터질까 봐 무서워서요 ㄷㄷ)
>대신 **기본값은 `False`** 로 두고, 필요할 때마다 엔지니어가 직접 **CLI로 Backfill** 하는 것이 훨씬 통제 가능하고 안전한 운영 방식입니다.
