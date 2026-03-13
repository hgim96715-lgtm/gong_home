---
aliases:
  - Docker Compose
  - Docker setup
tags:
  - Project
related:
  - "[[PostgreSQL_Setup]]"
  - "[[00_Seoul Station Real-Time Train Project|train-project]]"
  - "[[00_Docker_HomePage]]"
  - "[[Kafka_CLI_Cheatsheet]]"
  - "[[SQL_DDL_Create]]"
  - "[[04_Spark_Streaming]]"
---
# 🐳 STEP 1. Docker Compose 기초 세팅 


###  Kafka + PostgreSQL 먼저, 나중에 하나씩 추가

---

## ⚠️ Bitnami 이미지 사용 불가 — 배경

2025년 9월 29일부터 Bitnami 가 Broadcom 인수 후 버전별 이미지를 유료로 전환했다.

```
bitnami/kafka:3.7    → ❌ 유료 (Bitnami Secure Images 구독 필요)
bitnami/kafka:latest → ⚠️  무료지만 버전 고정 안됨 (운영 불안정)
bitnami/spark:3.5    → ❌ 동일하게 유료 전환
```

```
그래서 이 프로젝트는 Apache 공식 이미지를 사용한다.
apache/kafka:3.7.0  → ✅ 완전 무료, Apache 공식, 버전 고정 가능
apache/spark:3.5.0  → ✅ 완전 무료, Apache 공식, 버전 고정 가능
```

```text
bitnami vs apache 공식 이미지 환경변수 차이:

  Kafka
    bitnami  → KAFKA_CFG_* 접두사로 설정
    apache   → 접두사 없이 직접 설정값 사용

  Spark
    bitnami  → SPARK_MODE, SPARK_MASTER_URL 환경변수로 역할 지정
    apache   → command 로 직접 클래스 실행
               spark-class ...Master / ...Worker spark://host:port
```

> 단, bitnami → apache 공식 이미지는 환경변수 체계가 다르다. `KAFKA_CFG_` 접두사 없이 직접 설정값을 쓴다.

---

## 목표

```
이 단계에서 띄울 컨테이너:

Kafka (KRaft)   ← 메시지 큐
PostgreSQL      ← 저장소

✅ 두 개가 정상 실행되면 STEP 1 완료
나머지 (Spark, Airflow, Superset) 는 이후 단계에서 추가
```

---

## 사전 준비

```
✅ Docker Desktop 설치 확인
✅ 터미널에서 아래 명령어로 버전 확인

$ docker --version
$ docker compose version
```

---

## 폴더 구조 생성

```bash
mkdir seoul-train-realtime-project
cd seoul-train-realtime-project

mkdir -p producer spark airflow/dags superset postgres docs
```

```
seoul-train-realtime-project/
├── docker-compose.yml    ← 이 단계에서 작성
├── .env                  ← 이 단계에서 작성
├── postgres/
│   └── init.sql          ← 이 단계에서 작성
├── producer/             ← STEP 2~3
├── spark/                ← STEP 4
├── superset/             ← STEP 5
├── airflow/
│   └── dags/             ← STEP 6
└── docs/
```

---

## .env 파일

> API 키, 비밀번호 등 민감한 값은 `.env` 에 모아서 관리한다. `.gitignore` 에 반드시 추가해야 GitHub 에 올라가지 않는다.

```bash
# .env

# PostgreSQL
POSTGRES_USER=train_user 
POSTGRES_PASSWORD=train_password 
POSTGRES_DB=train_db
```

---

## postgres/init.sql

> PostgreSQL 컨테이너가 처음 실행될 때 자동으로 테이블을 생성한다.
> `docker-entrypoint-initdb.d/` 안에 마운트된 `.sql` 파일은 컨테이너 최초 실행 시 자동 실행.
> [한국철도공사_열차운행정보 참고](https://www.data.go.kr/data/15125762/openapi.do#)

```sql
-- 1. 여객열차 운행계획 (하루 1회 적재)
CREATE TABLE IF NOT EXISTS train_schedule (
    id                  SERIAL PRIMARY KEY,
    run_ymd             VARCHAR(8),
    trn_no              VARCHAR(20),
    dptre_stn_cd        VARCHAR(10),
    dptre_stn_nm        VARCHAR(50),
    arvl_stn_cd         VARCHAR(10),
    arvl_stn_nm         VARCHAR(50),
    trn_plan_dptre_dt    VARCHAR(50),
    trn_plan_dptre_dt    VARCHAR(50),
    data_type           VARCHAR(20),  -- 'schedule'
    created_at          TIMESTAMP DEFAULT NOW()
);

-- 2. 여객열차 운행 현황 (당일 시뮬레이션 전광판용)
-- 원본 API 필드 대신 파이썬의 run_estimated() 함수가 쏘는 필드들로 교체!
CREATE TABLE IF NOT EXISTS train_realtime (
    id                  SERIAL PRIMARY KEY,
    trn_no              VARCHAR(20),
    -- mrnt_nm             VARCHAR(50),  -- 노선명 ,추정이 너무 안맞아서 제거 -> train_delay 테이블에만 남김
    dptre_stn_nm        VARCHAR(50),  -- 출발역
    arvl_stn_nm         VARCHAR(50),  -- 도착역
    plan_dep            VARCHAR(10),  -- 계획 출발 (HH:MM)
    plan_arr            VARCHAR(10),  -- 계획 도착 (HH:MM)
    status              VARCHAR(100), -- 현재 상태 텍스트 (예: '운행 중 (50% 진행...)')
    progress_pct        INTEGER,      -- 진행률 (0~100)
    data_type           VARCHAR(20),  -- 'estimated'
    created_at          TIMESTAMP DEFAULT NOW()
);

-- 3. 지연 분석 (전날 계획 vs 실제 비교) — Superset 대시보드용
-- ENUM을 삭제하고 VARCHAR로 변경하여 "정시 (0분)" 같은 텍스트가 잘 들어가게 수정!
CREATE TABLE IF NOT EXISTS train_delay (
    id           SERIAL PRIMARY KEY,
    run_ymd      VARCHAR(8),       -- 운행 날짜 (전날)
    trn_no       VARCHAR(20),      -- 열차 번호
    mrnt_nm      VARCHAR(50),      -- 노선명 (경부선 등)
    dptre_stn_nm VARCHAR(50),      -- 출발역
    arvl_stn_nm  VARCHAR(50),      -- 도착역
    plan_dep     VARCHAR(10),      -- 계획 출발 HH:MM
    plan_arr     VARCHAR(10),      -- 계획 도착 HH:MM
    real_dep     VARCHAR(10),      -- 실제 출발 HH:MM
    real_arr     VARCHAR(10),      -- 실제 도착 HH:MM
    dep_delay    INTEGER,          -- 출발 지연 분 (양수=지연, 음수=조기)
    arr_delay    INTEGER,          -- 도착 지연 분
    dep_status   VARCHAR(50),      -- 상태 텍스트 (예: '정시 (0분)', '대폭지연 (+45분)')
    arr_status   VARCHAR(50),      -- 상태 텍스트
    data_type    VARCHAR(20),      -- 'delay_analysis'
    created_at   TIMESTAMP DEFAULT NOW()
);
```

## DataGrip 으로 PostgreSQL 연결

>`docker compose up -d` 후 DataGrip 에서 바로 테이블 확인 가능.

```text
DataGrip → New → Data Source → PostgreSQL

Host:     localhost
Port:     5433
Database: train_db
User:     train_user
Password: train_password
```

```text
연결 후 확인:
train_db → public → Tables
  ├── train_schedule   ← 당일 운행계획
  ├── train_realtime   ← 전날 실제 운행정보
  └── train_delay      ← 지연 분석 결과 (Superset 대시보드용)

테이블이 보이면 init.sql 이 정상 실행된 것
```

>⚠️ 테이블이 안 보이면? `init.sql` 은 컨테이너 **최초 실행 시에만** 자동 실행된다. 
>이미 볼륨이 있으면 실행 안 됨 → `docker compose down -v` 후 다시 `up`

>⚠️ **"user does not exist" 에러가 나면?** 로컬 5432 포트 충돌 가능성이 높다.
>`{bash}$ lsof -i :5432 # 로컬에서 5432 사용 중인지 확인`
>결과에 뭔가 뜨면 로컬 PostgreSQL 이 이미 5432 점유 중 
>→ `docker-compose.yml` 포트를 `5433:5432` 로 변경 후 DataGrip 도 5433 으로 수정

---

## docker-compose.yml

```yaml
services:

  # ── Kafka (KRaft 모드 — Apache 공식 이미지) ──────────────────
  kafka:
    image: apache/kafka:3.7.0
    container_name: train-kafka
    ports:
      - "9092:9092"
    env_file:
      - .env                                   # .env 파일을 컨테이너 OS 환경변수로 주입
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092   # ⚠️ localhost 금지 — Docker 서비스명 사용
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka_data:/var/lib/kafka/data
    networks:
      - train-network
    healthcheck:
      test: ["CMD", "/opt/kafka/bin/kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── PostgreSQL ─────────────────────────────────────────────
  postgres:
    image: postgres:16
    container_name: train-postgres
    ports:
      - "5433:5432"   # 맥북↔컨테이너: 5433 / 컨테이너끼리: 5432
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - train-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Spark Master ───────────────────────────────────────────
  spark-master:
    image: apache/spark:3.5.0
    container_name: train-spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    env_file:
      - .env                                   # KAFKA_BOOTSTRAP_SERVERS, POSTGRES_* 주입
    ports:
      - "8080:8080"   # Spark Web UI
      - "7077:7077"   # Spark Master 포트
      - "4040:4040" # Spark Jobs UI (실행 중인 작업 상세 화면) — master 에만 추가
    volumes:
      - ./spark:/opt/spark/apps                # consumer.py 마운트
    networks:
      - train-network

  # ── Spark Worker ───────────────────────────────────────────
  spark-worker:
    image: apache/spark:3.5.0
    container_name: train-spark-worker
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    env_file:
      - .env
    environment:
      - SPARK_WORKER_MEMORY=1G
      - SPARK_WORKER_CORES=1
    volumes:
      - ./spark:/opt/spark/apps                # consumer.py 마운트
    depends_on:
      - spark-master
    networks:
      - train-network

volumes:
  kafka_data:
  postgres_data:

networks:
  train-network:
    driver: bridge
```

```text
‼️‼️ networks 정의가 없으면:
  "service refers to undefined network train-network" 에러 발생
  → docker-compose.yml 맨 아래 networks 블록 반드시 추가

driver: bridge
  컨테이너끼리 이름으로 통신 가능 (kafka, postgres, spark-master)
  호스트 머신과는 ports 로 연결
```

```text
env_file 를 쓰는 이유:
  Docker Compose 는 .env 파일을 자동으로 읽어서
  컨테이너 안의 OS 환경변수로 주입해줌
  → consumer.py 에서 os.getenv("KAFKA_BOOTSTRAP_SERVERS") 바로 사용 가능
  → python-dotenv 패키지 불필요

KAFKA_ADVERTISED_LISTENERS = PLAINTEXT://kafka:9092 이어야 하는 이유:
  Kafka 가 클라이언트에게 광고하는 "나한테 이 주소로 접속해" 주소
  localhost 로 설정 시 다른 컨테이너에서 localhost = 자기 자신 → 연결 실패
  → 반드시 Docker 서비스명(kafka) 으로 설정

포트 정리:
  맥북 ↔ PostgreSQL  : 5433 (DataGrip 등)
  컨테이너 ↔ PostgreSQL: 5432 (Spark JDBC)
```



---

## 실행 및 확인

>[[Docker_Compose_Commands#④ docker compose ps — 상태 확인|docker 상태확인]] 참조 

```bash
# 컨테이너 실행
$ docker compose up -d --build

# 실행 상태 확인
$ docker compose ps
```

```
정상 실행 시:
NAME              STATUS
train-kafka       running (healthy)
train-postgres    running (healthy)
```

```bash
# Kafka 토픽 생성 (3개)
$ docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic train-schedule --partitions 1 --replication-factor 1

$ docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic train-realtime --partitions 1 --replication-factor 1

$ docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic train-delay --partitions 1 --replication-factor 1

# 토픽 확인
$ docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --list

# PostgreSQL 접속 확인
$ docker exec -it train-postgres psql -U train_user -d train_db
```

```bash
# 테이블 생성 확인
\dt
```

```bash
#  결과
train_db=# \dt
              List of relations
 Schema |      Name      | Type  |   Owner
--------+----------------+-------+------------
 public | train_delay    | table | train_user
 public | train_realtime | table | train_user
 public | train_schedule | table | train_user
(3 rows)
```

---

## 종료 / 초기화

>새로 init.sql에 추가한다면, 초기화 하고 다시 새로고침 해보면 나옴!

```bash
# 컨테이너 중지 (데이터 유지)
$ docker compose down

# 컨테이너 + 볼륨 전체 삭제 (초기화) 
$ docker compose down -v
```

---
✅ 완료되면 → [[02_API_Producer]] 로 이동