---
aliases:
  - Spark 설치
  - Spark Docker
  - PySpark Setup
  - 스파크 구축
  - Spark Custom
tags:
  - Spark
  - Docker
  - Setup
related:
  - "[[Apache_Spark_Concept]]"
  - "[[00_Airflow_HomePage]]"
---
## 개념 한 줄 요약

현재 사용 중인 Airflow 프로젝트 폴더 안에 **Spark 전용 격리 공간(`spark/`)** 을 만들어서, Airflow 설정과 섞이지 않게 깔끔하게 구축하는 방법이야.

---
## Directory Structure (스크린샷 기준) 

현재 `apache-airflow` 폴더 안에 **`spark` 폴더를 새로 생성**하고, 그 안에 설정 파일들을 몰아넣을 거야.

```bash
apache-airflow/              (현재 최상위 루트)
├── dags/                    (기존 폴더)
├── docker-compose.yaml      (Airflow용 - 건드리지 마!)
├── ...
└── spark/                   (✨ New! 여기서부터 Spark 영역)
    ├── docker-compose.yaml  (Spark 전용! Airflow꺼랑 다름)
    ├── spark-events/  # (자동생성) 이벤트 로그 저장소
    ├── working-dir/ # (자동생성) 작업 공간
    ├── apps/                (파이썬 스크립트 저장소)
    └── spark_conf/          (✨ 질문하신 폴더는 여기!), # ⭐️ 설정 파일 폴더 (직접 만들어야 함)
        ├── log4j2.properties # 로그 설정
        └── spark-defaults.conf # 스파크 기본 설정
```

**핵심:** Airflow의 `docker-compose.yaml`과 Spark의 `docker-compose.yaml`은 서로 다른 파일이야! 
Spark 파일은 꼭 **`spark/` 폴더 안**에 만들어야 해.

---
## Step-by-Step Setup (따라 하기)

### Step 1.**폴더 만들기** 

VS Code 탐색기에서 우클릭해서 아래 순서대로 폴더를 만들어.

1. `apache-airflow` 안에 `spark` 폴더 생성
2. `spark` 안에 `spark_conf` 폴더 생성
3. `spark` 안에 `apps` 폴더 생성

### Step 2. 설정 파일 생성 (빈 파일)

`spark/spark_conf/` 안에 아래 두 파일을 생성해. (내용은 비워둬도 돼, 일단 에러 방지용이야)

- log4j2.properties
- spark-defaults.conf

### Step 3. Docker Compose 작성

`spark/` 폴더 안에 `docker-compose.yaml` 파일을 만들고 코드를 붙여넣어.

**(코드 다시 확인: 경로가 조금 바뀌었어!)** 우리가 `spark` 폴더 안으로 들어왔으니, 경로는 `./` (현재 위치)를 기준으로 잡으면 돼.

---
## Execution (실행 방법)

이게 제일 중요해! **반드시 `spark` 폴더로 이동해서** 실행해야 해.

```bash
# 1. 스파크 폴더로 이동
cd spark

# 2. 실행 (여기서 docker-compose는 Airflow가 아니라 Spark꺼를 읽음)
docker-compose up -d
```

### 접속 확인 

- **주소:** `http://localhost:8081` (8080 아님!)
- 화면 상단 **URL**에 `spark://spark:7077`이라고 되어 있고, **Workers** **Alive: 1** 이라고 떠 있으면 성공! 


---
## FAQ (헷갈리는 점)

- **Q. "Airflow 켜져 있는데 이거 또 켜도 돼요?"**
    
    - **A. 당연하지!** 서로 다른 터미널에서 켜도 되고, 하나는 백그라운드(`-d`)로 돌고 있으니 상관없어.
    - Airflow는 8080, Spark는 8081이라서 서로 안 싸우고 잘 돌아갈 거야.
        
- **Q. "`spark_conf`를 `apache-airflow/config` 안에 넣으면 안 돼요?"**

    - **A. 비추천!** Airflow 설정(`airflow.cfg`)이랑 Spark 설정이 섞이면 나중에 멘붕 와.
    - "Spark 관련 파일은 무조건 `spark/` 폴더 안에 가둔다"는 원칙을 지켜줘.

