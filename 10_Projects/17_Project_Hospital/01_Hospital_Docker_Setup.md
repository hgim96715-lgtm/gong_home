---
aliases:
  - Docker
tags:
  - Project
related:
  - "[[00_Hospital_Project]]"
  - "[[Docker_Volumes]]"
  - "[[Docker_Compose_Setup]]"
  - "[[Docker_Compose_Commands]]"
  - "[[Docker_Image_vs_Container]]"
  - "[[Docker_Container_Interaction]]"
  - "[[Docker_Compose_Template]]"
---

# 🐳 STEP 1. Docker Compose 세팅

## 결정 사항

| 질문     | 결정                    | 이유                    |
| ------ | --------------------- | --------------------- |
| 기본 이미지 | Apache 공식 이미지         | Bitnami 유료 전환으로 사용 불가 |
| 초기 구성  | Kafka + PostgreSQL 먼저 | 단계별 동작 확인하면서 진행       |

```
서울역 프로젝트와 동일한 구조
컨테이너명 / DB명 / 테이블만 hospital 로 변경
```

> ⚠️ Bitnami 이미지 사용 불가 이유 → [[Docker_Image_vs_Container]] 참고

---

---

## 목표

```
이 단계에서 띄울 컨테이너:
  Kafka (KRaft)   ← 메시지 큐
  PostgreSQL      ← 저장소
  Spark
  Airflow
  Superset

✅ 전부 정상 실행되면 STEP 1 완료
```

---

---

## 폴더 구조

```bash
mkdir hospital-project
cd hospital-project

mkdir -p producer spark airflow/dags  postgres docs
```

```
hospital-project/
├── docker-compose.yml
├── .env
├── .gitignore
├── postgres/
│   └── init.sql
├── producer/             
├── spark/               
├── airflow/
│   └── dags/             
└── docs/
    └── README.md
```

---

---

## .gitignore

```gitignore
.env
producer/producer_state.json
```

---

---

## .env

```bash
# PostgreSQL
POSTGRES_USER=hospital_user
POSTGRES_PASSWORD=hospital_password
POSTGRES_DB=hospital_db

# API
HOSPITAL_API_KEY=여기에_공공데이터_API_키_붙여넣기

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:9092

# Superset
SUPERSET_SECRET_KEY=여기에_openssl_생성값_붙여넣기

# Airflow
AIRFLOW_SECRET_KEY=여기에_openssl_생성값_붙여넣기
AIRFLOW_SMTP_USER=본인@gmail.com
AIRFLOW_SMTP_PASSWORD=앱비밀번호16자리
```

```bash
# SUPERSET_SECRET_KEY / AIRFLOW_SECRET_KEY 생성
openssl rand -base64 42
# echo "SECRET_KEY=$(openssl rand -base64 42)" >> .env
```

>[[Linux_OpenSSL]] 참고 

---
---
## postgres/init.sql

```sql
-- Step 2 에서 테이블 설계 후 작성 ->데이터가 복잡 양이 많이 분리 
-- 지금은 파일만 만들어두기 (빈 파일)
```

> API 응답 컬럼 확인 후 테이블 설계 → [[02_Hospital_Data_Structure]] 에서 진행

---

---

## docker-compose.yml

```yaml
services:

  # ── Kafka (KRaft 모드) ─────────────────────────────────────
  kafka:
    image: apache/kafka:3.7.0
    container_name: hospital-kafka
    ports:
      - "9093:9092"
    env_file:
      - .env
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka_data:/var/lib/kafka/data
    networks:
      - hospital-network
    healthcheck:
      test: ["CMD", "/opt/kafka/bin/kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── PostgreSQL ─────────────────────────────────────────────
  postgres:
    image: postgres:16
    container_name: hospital-postgres
    ports:
      - "5434:5432"
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - hospital-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Spark Master ───────────────────────────────────────────
  spark-master:
    image: apache/spark:3.5.0
    container_name: hospital-spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "8083:8080"   # Spark Web UI
      - "7078:7077"   # Spark Master
      - "4041:4040"   # Spark Jobs UI
    volumes:
      - ./spark:/opt/spark/apps
    networks:
      - hospital-network

  # ── Spark Worker ───────────────────────────────────────────
  spark-worker:
    image: apache/spark:3.5.0
    container_name: hospital-spark-worker
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    environment:
      - SPARK_WORKER_MEMORY=2G # 1G → 2G (Worker Lost 에러 방지)
      - SPARK_WORKER_CORES=2
    volumes:
      - ./spark:/opt/spark/apps
    depends_on:
      - spark-master
    networks:
      - hospital-network

  # ── Superset ───────────────────────────────────────────────
  superset:
    image: apache/superset:3.1.0
    container_name: hospital-superset
    ports:
      - "8089:8088"
    environment:
      - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
    volumes:
      - superset_data:/app/superset_home
    networks:
      - hospital-network
    depends_on:
      - postgres

  # ── Airflow ────────────────────────────────────────────────
  airflow:
    image: apache/airflow:2.9.1
    container_name: hospital-airflow
    depends_on:
      - postgres
      - kafka
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
      AIRFLOW__CORE__FERNET_KEY: ''
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
      AIRFLOW__WEBSERVER__SECRET_KEY: ${AIRFLOW_SECRET_KEY}
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      AIRFLOW__SMTP__SMTP_HOST: 'smtp.gmail.com'
      AIRFLOW__SMTP__SMTP_STARTTLS: 'True'
      AIRFLOW__SMTP__SMTP_SSL: 'False'
      AIRFLOW__SMTP__SMTP_PORT: '587'
      AIRFLOW__SMTP__SMTP_USER: ${AIRFLOW_SMTP_USER}
      AIRFLOW__SMTP__SMTP_PASSWORD: ${AIRFLOW_SMTP_PASSWORD}
      AIRFLOW__SMTP__SMTP_MAIL_FROM: ${AIRFLOW_SMTP_USER}
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./producer:/opt/airflow/producer
    ports:
      - "8084:8080"
    networks:
      - hospital-network
    command: >
      bash -c "
        pip3 install kafka-python requests &&
        airflow db migrate &&
        airflow users create \
          --username admin \
          --password admin \
          --firstname Admin \
          --lastname User \
          --role Admin \
          --email admin@example.com || true &&
        airflow scheduler &
        airflow webserver
      "

volumes:
  kafka_data:
  postgres_data:
  superset_data:

networks:
  hospital-network:
    driver: bridge
```

```
포트 정리 (train 프로젝트와 동시 실행 가능):
  8083  Spark Web UI      (train: 8080)
  8084  Airflow Web UI    (train: 8082)
  8089  Superset Web UI   (train: 8088)
  7078  Spark Master      (train: 7077)
  5434  PostgreSQL        (train: 5433)
  9093  Kafka             (train: 9092)
```

---
---

## 실행 및 확인

```bash
# 컨테이너 실행
docker compose up -d

# 상태 확인
docker compose ps
```

```
정상 실행 시:
NAME                    STATUS
hospital-kafka          running (healthy)
hospital-postgres       running (healthy)
hospital-spark-master   running
hospital-spark-worker   running
hospital-superset       running
hospital-airflow        running
```

## 📊 Kafka 토픽 설계

```
er-realtime    5분마다   전국 응급실 가용병상 동적 
er-hospitals   1일 1회   병원 기본정보 (정적)
```

```bash
# Kafka 토픽 생성
docker exec -it hospital-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --create \
  --topic er-realtime --partitions 1 --replication-factor 1

docker exec -it hospital-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --create \
  --topic er-hospitals --partitions 1 --replication-factor 1

# 토픽 확인
docker exec -it hospital-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list
```

---

---

## 트러블슈팅

| 증상              | 원인                             | 해결                                           |
| --------------- | ------------------------------ | -------------------------------------------- |
| 5433 포트 충돌      | 로컬 PostgreSQL 이 5432 점유        | `lsof -i :5432` 확인 후 포트 변경                   |
| 테이블 안 보임        | 볼륨이 이미 있어서 init.sql 미실행        | `docker compose down -v` 후 재실행               |
| Airflow DAG 안 뜸 | `users create` 실패 → `&&` 체인 끊김 | <code>users create ...\|\| true && </code>확인 |


✅ 완료되면 → [[02_Hospital_Data_Structure]] 으로 이동