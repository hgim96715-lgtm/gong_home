---
aliases:
  - Spark Kafak Docker
  - Docker Compose
tags:
  - Spark
  - Kafka
  - Docker
related:
  - "[[Spark_Streaming_Socket_Boilerplate]]"
  - "[[Apache_Kafka_Concept]]"
  - "[[Spark_Kafka_Docker_Setup]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 스파크 + 카프카(KRaft) 통합 실습 환경

스파크(Spark) 클러스터와 카프카(Kafka)를 한 번에 실행하는 Docker Compose 설정입니다.
**Zookeeper 없이(KRaft 모드)** 실행되므로 가볍고 설정이 간편합니다.

* **Spark:** 3.5.1 (Master 1, Worker 1, Jupyter Lab)
* **Kafka:** 3.7.0 (Official Apache Image, KRaft Mode)

---
##  `docker-compose.yml` (최종본)

`bitnami` 이미지 대신 인증 문제가 없는 **공식 `apache/kafka` 이미지**를 사용합니다.

>Bitnami Kafka는 이제 GHCR 전용이고,  GHCR는 docker login 없이는 pull을 거부하고 있어서 Docker Hub에 이미지가 안보인다!

```yaml
services:
  # 1. Spark Master (Cluster Manager)
  spark-master:
    build: .
    container_name: spark-master
    command: >
      /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "9090:8080" # Spark Master Web UI
      - "7077:7077" # Internal Communication
    networks:
      - spark-net

  # 2. Spark Worker (Executor)
  spark-worker:
    build: .
    container_name: spark-worker
    command: >
      /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    depends_on:
      - spark-master
    volumes:
      - ./data:/workspace/data
    networks:
      - spark-net

  # 3. Jupyter Lab (Driver & Client)
  jupyter:
    build: .
    container_name: spark-jupyter
    command: >
      jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root --NotebookApp.token=''
    volumes:
      - ./notebooks:/workspace # 노트북 저장소
      - ./data:/workspace/data # 데이터 저장소
    ports:
      - "8888:8888" # Jupyter Lab 접속
      - "4040:4040" # Spark Job UI
    environment:
      - SPARK_MASTER=spark://spark-master:7077
    depends_on:
      - spark-master
    networks:
      - spark-net

  # 4. Kafka Broker (KRaft Mode - No Zookeeper)
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      # KRaft 및 통신 설정
      - KAFKA_NODE_ID=1
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      # 복제본 및 파티션 기본 설정 (실습용 1개)
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1
      - KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1
      - KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
      - KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=0
      - KAFKA_NUM_PARTITIONS=1
    networks:
      - spark-net

networks:
  spark-net:
    driver: bridge
```

---
## 실행 및 접속 방법 

### ① 실행하기

터미널에서 파일이 있는 폴더로 이동 후 실행합니다.

```bash
docker-compose down  # 기존 컨테이너 정리
docker-compose up -d --build # 빌드 및 실행
```

### ② 카프카 설치 확인 (토픽 생성)

카프카 컨테이너에 접속해서 토픽이 잘 만들어지는지 테스트합니다.

```bash
# 1. 카프카 컨테이너 내부로 진입
docker exec -it kafka bash

# 2. 토픽 생성 (이름: test-topic)
/opt/kafka/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server kafka:9092

# 3. 토픽 목록 확인
/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server kafka:9092
```


### ③ 파이썬(Spark) 연결 주소 

스파크 코드에서 카프카에 접속할 때는 아래 주소를 사용하세요.

- **Bootstrap Servers:** `kafka:9092`
- **이유:** Docker 네트워크 내부(`spark-net`)끼리는 컨테이너 이름(`kafka`)으로 통신하기 때문입니다.