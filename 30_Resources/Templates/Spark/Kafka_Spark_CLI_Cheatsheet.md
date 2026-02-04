---
aliases:
  - Kafka CLI
  - Spark Submit
  - 카프카 명령어
  - 도커 카프카
tags:
  - Kafka
  - Spark
  - Docker
  - CLI
  - Snippet
description: Docker 환경에서 자주 쓰는 카프카 생성/전송/구독 및 스파크 실행 명령어 모음
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Streaming_JSON_ETL_Project]]"
---
##  Kafka & Spark CLI 치트 시트

> **사용법:** 터미널을 열고 필요한 명령어를 복사(`Cmd+C`)해서 붙여넣기(`Cmd+V`) 하세요.

###  카프카 토픽 관리 (Topic)

**토픽 생성 (Create)**

```bash
# 'test-topic' 이라는 이름의 토픽 생성
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh \
  --create \
  --topic test-topic \
  --bootstrap-server kafka:9092 \
  --replication-factor 1 \
  --partitions 1
```


### **토픽 목록 확인 (List)**

- 현재 생성된 모든 우체통(토픽) 이름을 확인합니다.

```bash
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh \
  --list \
  --bootstrap-server kafka:9092
```

----
### 스파크 실행 (Spark Submit)

**카프카 패키지 포함 실행 (가장 중요!)**

- `--packages`: 카프카와 통신하기 위한 라이브러리를 Maven에서 다운로드합니다.
- **주의:** 스파크 버전(`3.5.1`)과 스칼라 버전(`2.12`)이 환경과 일치해야 합니다.

```bash
/opt/spark/bin/spark-submit \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1 \
  kafka_stateful.py
```

---
### 메시지 테스트 (Pub/Sub)

**프로듀서 실행 (메시지 보내기)**

- 실행 후 터미널에 글자를 치고 엔터를 누르면 메시지가 전송됩니다.
- `pra-json`은 보내려는 토픽 이름입니다.

```bash
docker exec -it kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server kafka:9092 \
  --topic pra-json
```

### **컨슈머 실행 (메시지 확인하기)**

- 실시간으로 들어오는 메시지를 눈으로 확인합니다.
- `transformed`는 확인하려는 토픽 이름입니다.

```bash
docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic transformed \
  --property print.value=true
```

