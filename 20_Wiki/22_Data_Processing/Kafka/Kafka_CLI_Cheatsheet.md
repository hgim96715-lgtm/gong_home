---
aliases:
  - 카프카 CLI
  - kafka 명령어
  - 토픽 생성
  - kafka-topics
  - kafka-console-producer
  - kafka-console-consumer
  - bootstrap-server
  - from-beginning
  - 토픽 목록
  - LAG
tags:
  - Kafka
related:
  - "[[00_Kafka_HomePage]]"
  - "[[Docker_Compose_Setup]]"
---

# Kafka_CLI_Cheatsheet

## 개념 한 줄 요약

> **"Kafka 를 터미널에서 직접 조작하는 명령어 모음."** Docker 안에서 실행하기 때문에 `docker exec 컨테이너명` 이 항상 앞에 붙는다.

---

---

# ① 명령어 구조 이해 — 외우지 말고 조립하기

```
docker exec 컨테이너명  /opt/kafka/bin/스크립트.sh  --옵션들
    ↑                        ↑                          ↑
Docker 컨테이너 안에서   Kafka 가 설치된 경로의      실제 작업 내용
실행하라                 실행 파일
```

```bash
# 실제 예시 분해
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create \
    --topic train-realtime \
    --partitions 1 \
    --replication-factor 1
```

```
--bootstrap-server localhost:9092   Kafka 브로커 주소 (항상 붙임)
--create                            어떤 작업을 할 것인가 (토픽 생성)
--topic train-realtime              대상 토픽 이름
--partitions 1                      파티션 수
--replication-factor 1              복제 수 (브로커 1대면 반드시 1)
```

> `--bootstrap-server localhost:9092` 는 모든 명령어에 항상 붙는다. 컨테이너 안에서 실행하기 때문에 localhost 가 컨테이너 자신을 가리킨다.

---

---

# ② kafka-topics.sh — 토픽 관리

## 토픽 생성 (--create)

```bash
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create \
    --topic train-realtime \
    --partitions 1 \
    --replication-factor 1
```

## 토픽 목록 확인 (--list)

```bash
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --list
```

```
출력 예시:
train-realtime
train-schedule
```

## 토픽 상세 정보 확인 (--describe)

```bash
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --describe \
    --topic train-realtime
```

```
출력 예시:
Topic: train-realtime   PartitionCount: 1   ReplicationFactor: 1
    Partition: 0   Leader: 1   Replicas: 1   Isr: 1

읽는 법:
PartitionCount    파티션이 몇 개인지
ReplicationFactor 복제본이 몇 개인지
Leader            현재 이 파티션을 담당하는 브로커 ID
Isr               In-Sync Replicas (동기화된 복제본 목록)
```

## 토픽 삭제 (--delete)

```bash
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --delete \
    --topic train-realtime
```

## kafka-topics.sh 옵션 정리

|옵션|설명|
|---|---|
|`--create`|토픽 생성|
|`--list`|토픽 목록 조회|
|`--describe`|토픽 상세 정보|
|`--delete`|토픽 삭제|
|`--topic 이름`|대상 토픽 지정|
|`--partitions N`|파티션 수|
|`--replication-factor N`|복제 수 (브로커 수 이하)|

---

---

# ③ kafka-console-producer.sh — 메시지 전송

```bash
docker exec -it train-kafka \
    /opt/kafka/bin/kafka-console-producer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime
```

```
실행 후 > 프롬프트가 뜨면 입력 시작
> 첫 번째 메시지
> 두 번째 메시지
> Ctrl+C 로 종료
```

## Key-Value 형태로 전송 (--property parse.key=true)

```bash
docker exec -it train-kafka \
    /opt/kafka/bin/kafka-console-producer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime \
    --property "key.separator=:" \
    --property "parse.key=true"

# 입력 형태: 키:값
> train-001:{"station":"서울","time":"09:00"}
> train-002:{"station":"수원","time":"09:30"}
```

---

---

# ④ kafka-console-consumer.sh — 메시지 수신

## 기본 소비 (새 메시지부터)

```bash
docker exec -it train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime
```

## 처음부터 전부 읽기 (--from-beginning) ⭐️

```bash
docker exec -it train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime \
    --from-beginning
```

```
--from-beginning 없으면: 실행 이후 새로 들어오는 메시지만 수신
--from-beginning 있으면: 토픽에 저장된 처음 메시지부터 전부 수신
                         테스트 / 디버깅 시 필수
```

## 메타데이터 함께 출력 (--property print.key / print.timestamp)

```bash
docker exec -it train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime \
    --from-beginning \
    --property print.key=true \
    --property print.timestamp=true
```

```
출력 예시:
CreateTime:1710000000000    train-001    {"station":"서울"}
```

## N개만 읽고 종료 (--max-messages)

```bash
docker exec train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime \
    --from-beginning \
    --max-messages 10
```

## kafka-console-consumer.sh 옵션 정리

|옵션|설명|
|---|---|
|`--from-beginning`|처음 메시지부터 전부 읽기|
|`--max-messages N`|N개만 읽고 종료|
|`--group 그룹명`|컨슈머 그룹 지정|
|`--property print.key=true`|키 함께 출력|
|`--property print.timestamp=true`|타임스탬프 함께 출력|

---

---

# ⑤ 컨슈머 그룹 확인 — kafka-consumer-groups.sh

```bash
# 컨슈머 그룹 목록
docker exec train-kafka \
    /opt/kafka/bin/kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --list

# 그룹 상세 (LAG 확인)
docker exec train-kafka \
    /opt/kafka/bin/kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --describe \
    --group 그룹명
```

```
LAG = 아직 읽지 못한 메시지 수
LAG = 0     -> 컨슈머가 모든 메시지를 소화함 (정상)
LAG > 0     -> 컨슈머가 프로듀서 속도를 못 따라감 (병목)
```

---

---

# 자주 쓰는 명령어 한눈에 보기

```bash
# 토픽 생성
docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create --topic 토픽명 --partitions 1 --replication-factor 1

# 토픽 목록
docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --list

# 토픽 상세
docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --describe --topic 토픽명

# 메시지 전송 (인터랙티브)
docker exec -it train-kafka /opt/kafka/bin/kafka-console-producer.sh \
    --bootstrap-server localhost:9092 --topic 토픽명

# 메시지 수신 (처음부터)
docker exec -it train-kafka /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 --topic 토픽명 --from-beginning
```

> `-it` 는 인터랙티브 터미널 옵션. 키보드 입력이 필요한 producer / consumer 실행 시 반드시 붙인다. 스크립트 안에서 자동 실행할 때는 `-it` 없이 사용.