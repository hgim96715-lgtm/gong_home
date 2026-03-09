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
---
# 🐳 STEP 1. Docker Compose 기초 세팅 


###  Kafka + PostgreSQL 먼저, 나중에 하나씩 추가

---

## ⚠️ Bitnami 이미지 사용 불가 — 배경

2025년 9월 29일부터 Bitnami 가 Broadcom 인수 후 버전별 이미지를 유료로 전환했다.

```
bitnami/kafka:3.7    → ❌ 유료 (Bitnami Secure Images 구독 필요)
bitnami/kafka:latest → ⚠️  무료지만 버전 고정 안됨 (운영 불안정)
```

```
그래서 이 프로젝트는 Apache 공식 이미지를 사용한다.
apache/kafka:3.7.0  → ✅ 완전 무료, Apache 공식, 버전 고정 가능
```

> 단, bitnami → apache 공식 이미지는 환경변수 체계가 다르다. `KAFKA_CFG_` 접두사 없이 직접 설정값을 쓴다.

---

## 목표

```
Kafka (KRaft)   ← 메시지 큐
PostgreSQL      ← 저장소
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
└── postgres/
    └── init.sql          ← 이 단계에서 작성
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
-- 여객열차 운행계획 (예정 시각)
CREATE TABLE IF NOT EXISTS train_schedule (
    id                  SERIAL PRIMARY KEY,
    run_ymd             VARCHAR(8),
    trn_no              VARCHAR(20),
    dptre_stn_cd        VARCHAR(10),
    dptre_stn_nm        VARCHAR(50),
    arvl_stn_cd         VARCHAR(10),
    arvl_stn_nm         VARCHAR(50),
    trn_plan_dptre_dt   VARCHAR(20),
    trn_plan_arvl_dt    VARCHAR(20),
    created_at          TIMESTAMP DEFAULT NOW()
);

-- 여객열차 운행정보 (실제 시각)
CREATE TABLE IF NOT EXISTS train_realtime (
    id                  SERIAL PRIMARY KEY,
    run_ymd             VARCHAR(8),
    trn_no              VARCHAR(20),
    trn_run_sn          VARCHAR(20),
    stn_cd              VARCHAR(10),
    stn_nm              VARCHAR(50),
    mrnt_cd             VARCHAR(10),
    mrnt_nm             VARCHAR(50),
    uppln_dn_se_cd      VARCHAR(5),
    stop_se_cd          VARCHAR(5),
    stop_se_nm          VARCHAR(20),
    trn_dptre_dt        VARCHAR(20),
    trn_arvl_dt         VARCHAR(20),
    created_at          TIMESTAMP DEFAULT NOW()
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
  ├── train_schedule   ← 운행계획 테이블
  └── train_realtime   ← 운행정보 테이블

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
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka_data:/var/lib/kafka/data
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
      - "5433:5432"  # 로컬 5432 충돌로 인해 호스트 포트를 5433 으로 변경
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  kafka_data:
  postgres_data:
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
# Kafka 토픽 생성 (열차 데이터용)
$ docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create \
    --topic train-realtime \
    --partitions 1 \
    --replication-factor 1

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
 public | train_realtime | table | train_user
 public | train_schedule | table | train_user
(2 rows)
```

---

## 종료 / 초기화

```bash
# 컨테이너 중지 (데이터 유지)
$ docker compose down

# 컨테이너 + 볼륨 전체 삭제 (초기화)
$ docker compose down -v
```

---

## 완료 체크

- [ ] `docker compose ps` 에서 두 컨테이너 모두 `healthy`
- [ ] `train-realtime` Kafka 토픽 생성 확인
- [ ] PostgreSQL 접속 후 `\dt` 로 테이블 2개 확인

✅ 완료되면 → [[02_API_Producer]] 로 이동