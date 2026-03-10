## 1. Concept & Architecture (기초)

- [[Apache_Kafka_Concept]] : 카프카가 도대체 뭔가요? (`Pub/Sub`, `Producer`, `Consumer`, `Broker`, `메시지 큐`, `이벤트 스트리밍`)
- [[Kafka_Topic_Partition_Offset]] : 파티션과 오프셋, 병렬 처리의 핵심 ⭐️ (`Topic`, `Partition`, `Offset`, `병렬 처리`, `순서 보장`, `메시지 위치`)
- [[Kafka_Consumer_Group_Lag]] : 컨슈머 그룹과 랙(Lag) 관리 (`Consumer Group`, `Lag`, `group.id`, `파티션 할당`, `처리 지연 모니터링`)
- [[Kafka_Replication_ISR]] : 데이터 복제와 장애 복구 원리 (`Replication Factor`, `ISR`, `Leader`, `Follower`, `장애 복구`, `데이터 유실 방지`)
- [[Docker_Host_vs_Internal_Network]] : `localhost` vs `Container_Name` 완벽 정리 (`KAFKA_ADVERTISED_LISTENERS`, `외부 접속`, `내부 통신`, `브릿지 네트워크`, `포트 바인딩`)

---

## 2. CLI & Operations (도구)

- [[Kafka_CLI_Cheatsheet]] : 터미널 필수 명령어 모음 (`kafka-topics.sh`, `kafka-console-producer.sh`, `kafka-console-consumer.sh`, `--list`, `--describe`, `--from-beginning`)
- [[Kafka_Docker_Compose_Setup]] : Docker 환경 구축 및 트러블슈팅 (`docker-compose`, `Zookeeper`, `Broker`, `환경변수`, `포트 설정`, `트러블슈팅`)

---

## 3. Python Development (코딩)

- [[Kafka_Python_Producer]] : 데이터 전송 기초 (`confluent_kafka`, `KafkaProducer`, `produce`, `flush`, `key/value`, `Hello World`)
- [[Kafka_Python_Consumer]] : 데이터 수신 기초 (`KafkaConsumer`, `subscribe`, `poll`, `무한 루프`, `auto_offset_reset`, `commit`)
- [[Kafka_Python_Serialization]] : JSON 데이터 직렬화/역직렬화 패턴 (`json.dumps`, `json.loads`, `encode`, `decode`, `value_serializer`, `직렬화 패턴`)
- [[Kafka_Error_Handling_Retry]] : 연결 실패 및 에러 처리 (`try/except`, `재시도`, `좀비 패턴`, `KafkaException`, `백오프`, `연결 실패 복구`)

---

## 4. PyFlink Integration (연동)

- [[PyFlink_Kafka_Source_Sink]] : Flink와 Kafka 연결하기 (`FlinkKafkaConsumer`, `FlinkKafkaProducer`, `Source`, `Sink`, `connector jar`, `체크포인트`)
- [[Flink_Kafka_SerializationSchema]] : Flink에서의 데이터 스키마 정의 (`SerializationSchema`, `DeserializationSchema`, `JSONRowSerializationSchema`, `TypeInformation`)

---

## 5. Data Lake Integration (데이터 레이크 연동)

> "카프카의 실시간 데이터를 모아서 영구 보존용 창고(MinIO/S3)에 쌓아보자"

- [[Kafka_to_MinIO_DataLake]] : Kafka Consumer와 Pandas를 이용한 Parquet 배치 저장 (`s3fs`, `Parquet`, `배치 저장`, `MinIO`, `S3 호환`, `pandas.to_parquet`, `통역사 설정`)