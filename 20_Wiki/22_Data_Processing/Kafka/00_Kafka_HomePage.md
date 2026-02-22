
## 1. Concept & Architecture (기초)

- [[Apache_Kafka_Concept]] : 카프카가 도대체 뭔가요? (Pub/Sub)
- [[Kafka_Topic_Partition_Offset]] : 파티션과 오프셋, 병렬 처리의 핵심 ⭐️
- [[Kafka_Consumer_Group_Lag]] : 컨슈머 그룹과 랙(Lag) 관리
- [[Kafka_Replication_ISR]] : 데이터 복제와 장애 복구 원리
- [[Docker_Host_vs_Internal_Network]] : `localhost` vs `Container_Name` (외부 접속과 내부 통신 차이 완벽 정리)

## 2. CLI & Operations (도구)

- [[Kafka_CLI_Cheatsheet]] : 터미널 필수 명령어 모음 (생성, 조회, 테스트) 
- [[Kafka_Docker_Compose_Setup]] : Docker 환경 구축 및 트러블슈팅

## 3. Python Development (코딩)

- [[Kafka_Python_Producer_Basic]] : 데이터 전송 기초 (Hello World)
- [[Kafka_Python_Consumer_Basic]] : 데이터 수신 기초 (무한 루프)
- [[Kafka_Python_Serialization]] : JSON 데이터 직렬화/역직렬화 패턴
- [[Kafka_Error_Handling_Retry]] : 연결 실패 및 에러 처리 (좀비 패턴)

## 4. PyFlink Integration (연동)

- [[PyFlink_Kafka_Source_Sink]] : Flink와 Kafka 연결하기
- [[Flink_Kafka_SerializationSchema]] : Flink에서의 데이터 스키마 정의

## 5. Data Lake Integration (데이터 레이크 연동)

> "카프카의 실시간 데이터를 모아서 영구 보존용 창고(MinIO/S3)에 쌓아보자"

- [[Kafka_to_MinIO_DataLake]] : Kafka Consumer와 Pandas를 이용한 Parquet 배치 저장 (s3fs 통역사 설정)