---
aliases:
  - CLI
  - Command Line
  - 터미널 명령어
  - dags test
  - tasks test
tags:
  - Airflow
  - Operations
  - Tools
  - Debugging
related:
  - "[[Catchup_and_Backfill]]"
---
## 개념 한 줄 요약

**Airflow CLI**는 웹 UI(마우스)를 쓰지 않고, **검은색 터미널 창(키보드)** 에서 명령어를 쳐서 Airflow를 조종하는 리모컨이야.

---
##  왜 필요한가 (Why)

 **UI의 한계:** 
 - "DAG 켜기/끄기", "로그 보기" 정도만 가능하고 디테일한 제어가 힘들어.
 
**CLI의 장점 (Why Use):**
- **Backfill (과거 재처리):** "3월 1일부터 5일까지 다시 돌려줘" 같은 정교한 작업은 CLI가 필수야.
-  **Debugging (미리 테스트):** 배포 전에 `test` 명령어로 에러가 나는지 미리 볼 수 있어. (**개발 속도 2배 향상 🚀**)
-  **Automation:** 쉘 스크립트로 반복 작업을 자동화할 수 있어.

---
## Practical Context (접속 위치 & 방법)

가장 많이 하는 착각: "CLI 명령어 치려면 `docker-compose.yml` 있는 폴더로 가야 하나요?"

### A. 서버를 켤 때 (`docker-compose up`) 

* **위치:** 반드시 `docker-compose.yml` 파일이 있는 폴더여야 해.
* **이유:** 설계도(YAML)를 보고 건물을 지어야 하니까.

### B. 명령어를 쏠 때 (`docker exec` / Airflow CLI) 

* **위치:** **아무 데나 상관없어!** (내 홈 디렉토리 `~` 여도 됨)
* **이유:** 이미 지어진 건물(컨테이너)의 **이름(Name)** 만 알면, 어디서든 무전(Exec)을 칠 수 있거든.

> **실전 요약:**
> 1. 터미널 켜고 `cd`로 이동할 필요 없이,
> 2. 그냥 `docker exec -it <컨테이너이름> airflow ...` 날리면 끝!

**Local (가상환경)** 여긴 **가상환경을 켜는 게** 중요해.

```bash
# 가상환경 활성화
source .venv/bin/activate
airflow dags list
```

---
##  Core Commands (필수 명령어 3대장) ⭐️

> **💡 Docker 유저 필독:** 아래 명령어 앞에 `docker exec -it <컨테이너명>` 만 붙이면 돼.

## A. 조회하기 (`list`)

"지금 등록된 DAG가 뭐 뭐 있지?"

```bash
# Docker 예시 (밖에서 쏠 때)
docker exec -it my-scheduler airflow dags list

# Local 예시
airflow dags list
```

## B. 테스트하기 (`test`) - 실무 필수!

"이 DAG 지금 실행하면 에러 안 나나? (DB 기록 없이 로그만 확인)"

```bash
# docker 예시(밖에서 쏠때)
# 문법: docker exec -it <컨테이너명> airflow dags test <DAG_ID> <Execution_Date>
docker exec -it my-scheduler airflow dags test my_first_dag 2024-01-01

# Local 문법: airflow dags test <DAG_ID> <Execution_Date>
airflow dags test my_first_dag 2024-01-01
```

**💡 Q. 날짜(Execution Date)에는 뭘 넣어야 해?**  **A. "네가 처리하고 싶은 데이터의 날짜" (오늘 날짜 X)**
- Airflow에게 **"너는 지금 2024년 1월 1일로 시간여행을 온 거야"** 라고 최면을 거는 거야.
- 코드 안의 `{{ ds }}` 변수가 이 날짜로 바뀌어서 쿼리가 실행돼.

### C. 재처리하기 (Backfill) - 문법 사전 📖

> 🚨 **자세한 백필 전략과 시나리오(Clear vs Create)는 [[Catchup_and_Backfill]] 문서 참고!**
> 여기서는 **명령어 스펠링**만 확인해.

**1. 백필 생성 (Create)**

```bash
# 문법: air backfill create <DAG_ID> --from-date <시작> --to-date <끝>
air backfill create backfilling_sample \
    --from-date 2026-01-02 \
    --to-date 2026-01-05
```

**2. 결과 확인 (List Runs)**

```bash
# 문법: air dags list-runs <DAG_ID>
air dags list-runs backfilling_sample
```

**3. 실패 태스크 재실행 (Clear)**

```bash
# 문법: air tasks clear <DAG_ID> --only-failed ...
air tasks clear backfilling_sample \
    --only-failed \
    --start-date 2026-01-02 \
    --end-date 2026-01-05
```

**4. 실수로 만든 DAG Run 삭제 (Delete)**

```bash
# 문법: air dags delete-dag-run <DAG_ID> --run-id <ID>
air dags delete-dag-run backfilling_sample --run-id <복사한_ID>
```

---
#### **[Legacy Way] Airflow 2.9 이하 (구식)**

아직 많이 쓰이지만, 터미널 끄면 백필도 멈추는 방식.

```bash
# airflow dags backfill <DAG_ID> -s <시작> -e <끝>
airflow dags backfill my_dag -s 2024-01-01 -e 2024-01-03
```

---
## Troubleshooting & Tips

#### **1. `zsh: command not found: airflow`**

- **원인:** 내 맥북(Host)엔 Airflow가 없어. Docker 컨테이너 **안**에 있지.
- **해결:** `docker exec -it ...` 를 앞에 붙이거나 컨테이너 안으로 들어가.

#### **2. 매번 `docker exec...` 치기 귀찮아! (Alias 설정)**

**Step 1. 내 스케줄러 진짜 이름 찾기**

- 먼저  **실제 컨테이너 이름**을 복사해둬.

```bash
docker ps
# 예시: apache-airflow-airflow-scheduler-1 <-- 이거 복사!
```

**Step 2. 설정 파일 열기 (`.zshrc`)** 편한 에디터로 설정 파일을 열어.

```bash
# VS Code가 깔려 있다면 (추천)
code ~/.zshrc

# 터미널 에디터를 쓴다면
vi ~/.zshrc
```

**Step 3. 별명 등록하기 (맨 아래에 추가)** 파일 맨 밑에 아래 한 줄을 추가하고 저장해. 

(주의: `apache-...` 부분에 아까 복사한 **네 컨테이너 이름**을 넣어야 해!)

```bash
alias air='docker exec -it apache-airflow-airflow-scheduler-1 airflow'
```

**Step 4. 적용하기 (중요! ✨)** 이걸 안 하면 터미널을 껐다 켜야 적용돼. 지금 당장 적용하려면:

```bash
source ~/.zshrc
```

**이제부터는 이렇게만 치면 끝!**

```bash
air dags list
air dags test my_dag 2024-01-01
```

**3. "왜 명령어가 자꾸 바뀌나요?"**

- **Legacy:** "DAG야, 당장 행동해!" (Action 중심)
- **Modern:** "백필 요청서(객체)를 만들어!" (Object 중심) -> 더 안정적인 운영을 위한 진화야.

