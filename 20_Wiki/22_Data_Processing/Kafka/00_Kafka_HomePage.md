

```
실시간 데이터를 안정적으로 흘리는 메시지 스트리밍 플랫폼
Producer → Broker → Consumer 로 데이터가 흐른다
```

---

---

## Level 0. 개념 잡기

```
카프카가 도대체 뭔가?
왜 단순 DB 저장이 아니라 메시지 큐를 쓰는가?
```

|노트|핵심 개념|
|---|---|
|[[Apache_Kafka_Concept]]|Pub/Sub / Producer / Consumer / Broker / 메시지 큐 / 이벤트 스트리밍|
|[[Kafka_Topic_Partition_Offset]]|Topic / Partition / Offset / 병렬 처리 / 순서 보장 / 메시지 위치|

---

---

## Level 1. 환경 세팅

```
Docker 로 Kafka 띄우고
localhost vs 컨테이너 이름 차이 이해
```

| 노트                                  | 핵심 개념                                                                                                            |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| [[Docker_Host_vs_Internal_Network]] | localhost vs 컨테이너명 / KAFKA_LISTENERS / KAFKA_ADVERTISED_LISTENERS / 브릿지 네트워크 / 포트 바인딩                            |
| [[Kafka_CLI_Cheatsheet]]            | kafka-topics.sh / kafka-console-producer.sh / kafka-console-consumer.sh / --list / --describe / --from-beginning |

---

---

## Level 2. Python 연동

```
Python 으로 Kafka 에 데이터 보내고 받기
직렬화 / 역직렬화 패턴
```

|노트|핵심 개념|
|---|---|
|[[Kafka_Python_Producer]]|KafkaProducer / produce / flush / key/value / acks / retries|
|[[Kafka_Python_Consumer]]|KafkaConsumer / subscribe / poll / 무한루프 / auto_offset_reset / commit|
|[[Kafka_Python_Serialization]]|json.dumps / json.loads / encode / decode / value_serializer / 직렬화 패턴|

---

---

## Level 3. 안정성

```
연결 실패 / 재시도 / 에러 처리
실제 운영에서 발생하는 문제들
```

|노트|핵심 개념|
|---|---|
|[[Kafka_Error_Handling_Retry]]|try/except / 재시도 / 좀비 패턴 / KafkaException / 백오프 / 연결 실패 복구|

---

---

## Level 4. 활용

```
Kafka 에서 읽어서 어디로 보낼 것인가
```

|노트|핵심 개념|
|---|---|
|[[Kafka_to_MinIO_DataLake]]|Consumer + Pandas / Parquet 배치 저장 / MinIO / S3 호환 / pandas.to_parquet|

---

---

## 프로젝트 적용

|노트|설명|
|---|---|
|[[03_Kafka_Producer]]|서울역 프로젝트 Producer|
|[[03_Hospital_Producer]]|Hospital 프로젝트 Producer|

---

---
