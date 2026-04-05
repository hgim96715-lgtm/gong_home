---
aliases:
  - docker-compose 템플릿
---
# Template_Docker_Compose

```
사용법:
  1. 이 파일 복사
  2. PROJECT_NAME / DB_NAME / DB_USER 전체 교체
  3. 필요없는 서비스 주석 처리 또는 삭제
  4. .env 값 채우기
```

---

---

## .env

```bash
# PostgreSQL
POSTGRES_USER=PROJECT_NAME_user
POSTGRES_PASSWORD=PROJECT_NAME_password
POSTGRES_DB=PROJECT_NAME_db

# API Key
PROJECT_API_KEY=여기에_API_키

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:9092

# Superset
SUPERSET_SECRET_KEY=여기에_openssl_생성값     # openssl rand -base64 42

# Airflow
AIRFLOW_SECRET_KEY=여기에_openssl_생성값       # openssl rand -base64 42
AIRFLOW_SMTP_USER=본인@gmail.com
AIRFLOW_SMTP_PASSWORD=앱비밀번호16자리
```

---

---

## .gitignore

```gitignore
.env
producer/producer_state.json
__pycache__/
*.pyc
```

---

---

## 폴더 구조

```bash
mkdir PROJECT_NAME
cd PROJECT_NAME
mkdir -p producer spark airflow/dags postgres docs
touch postgres/init.sql
touch .env
touch .gitignore
```

---

---

## docker-compose.yml

```yaml
services:

  # ── Kafka (KRaft 모드 — Apache 공식) ──────────────────────
  kafka:
    image: apache/kafka:3.7.0
    container_name: PROJECT_NAME-kafka
    ports:
      - "9092:9092"
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
      - PROJECT_NAME-network
    healthcheck:
      test: ["CMD", "/opt/kafka/bin/kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── PostgreSQL ─────────────────────────────────────────────
  postgres:
    image: postgres:16
    container_name: PROJECT_NAME-postgres
    ports:
      - "5433:5432"   # 로컬 PostgreSQL 과 충돌 방지
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - PROJECT_NAME-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Spark Master ───────────────────────────────────────────
  spark-master:
    image: apache/spark:3.5.0
    container_name: PROJECT_NAME-spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "8080:8080"   # Spark Web UI
      - "7077:7077"   # Spark Master
      - "4040:4040"   # Spark Jobs UI
    volumes:
      - ./spark:/opt/spark/apps
    networks:
      - PROJECT_NAME-network

  # ── Spark Worker ───────────────────────────────────────────
  spark-worker:
    image: apache/spark:3.5.0
    container_name: PROJECT_NAME-spark-worker
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    environment:
      - SPARK_WORKER_MEMORY=1G
      - SPARK_WORKER_CORES=1
    volumes:
      - ./spark:/opt/spark/apps
    depends_on:
      - spark-master
    networks:
      - PROJECT_NAME-network

  # ── Superset ───────────────────────────────────────────────
  superset:
    image: apache/superset:3.1.0
    container_name: PROJECT_NAME-superset
    ports:
      - "8088:8088"
    environment:
      - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
    volumes:
      - superset_data:/app/superset_home
    networks:
      - PROJECT_NAME-network
    depends_on:
      - postgres

  # ── Airflow ────────────────────────────────────────────────
  airflow:
    image: apache/airflow:2.9.1
    container_name: PROJECT_NAME-airflow
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
      - "8082:8080"   # 8080=Spark, 8088=Superset 충돌 방지
    networks:
      - PROJECT_NAME-network
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
  PROJECT_NAME-network:
    driver: bridge
```

---

---

## 포트 정리

|서비스|포트|URL|
|---|---|---|
|Spark Web UI|8080|http://localhost:8080|
|Airflow Web UI|8082|http://localhost:8082|
|Superset|8088|http://localhost:8088|
|Spark Jobs UI|4040|http://localhost:4040|
|PostgreSQL|5433|localhost:5433|
|Kafka|9092|localhost:9092|

---

---

## 프로젝트별 교체 목록

```
PROJECT_NAME 을 프로젝트 이름으로 전체 교체:
  서울역 프로젝트  → train
  Hospital 프로젝트 → hospital
  새 프로젝트      → 원하는 이름

container_name:
  PROJECT_NAME-kafka
  PROJECT_NAME-postgres
  PROJECT_NAME-spark-master
  PROJECT_NAME-spark-worker
  PROJECT_NAME-superset
  PROJECT_NAME-airflow

network:
  PROJECT_NAME-network

.env:
  POSTGRES_USER=PROJECT_NAME_user
  POSTGRES_DB=PROJECT_NAME_db
```

---

---
