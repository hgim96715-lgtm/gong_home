---
aliases:
  - PyFlink Sink
  - kafka Sink
  - Data Sink Pattern
tags:
  - PyFlink
related:
  - "[[PyFlink_Kafka_Source]]"
  - "[[PyFlink + Kafka 연동 완벽 가이드 ⭐️]]"
---
#  PyFlink Data Sink: 데이터를 최종 목적지로!

## 한줄 요약

**"Source가 수도꼭지라면, Sink는 배수구다."**
데이터 스트림의 끝점(Endpoint)으로, 처리된 데이터를 외부 시스템(Kafka, S3, DB 등)으로 내보내는 역할을 한다.

---
##  Why: 왜 필요한가?

* `print()`는 개발자 눈에만 보일 뿐, 데이터가 저장되지 않고 휘발됨.
* 가공된 데이터를 다른 팀이나 애플리케이션이 쓰려면 **저장소(Storage)** 나 **큐(Queue)** 로 보내줘야 함.

---
##  Supported Sinks (지원하는 목적지)

PyFlink는 다양한 커넥터를 지원합니다.
* **File:** HDFS, S3, Local File 
* **Messaging:** Kafka, RabbitMQ 
* **Database:** JDBC (MySQL, Postgres), Redis, Elasticsearch 
* **Socket:** 네트워크 포트로 전송 

---
## Practical Code: Kafka Source -> Kafka Sink 

 **"Kafka에서 읽어서(Source), 들어온 문자열에 " - Processed" 를 붙임 & 단어마다 앞글자 대문자 , 다시 Kafka로 저장(Sink)"** 하는 코드를 작성해보기 

### 🛠️ 준비물

1.  **JAR 파일:** 이미 받아둔 `flink-sql-connector-kafka-3.1.0-1.18.jar` 하나면 Source와 Sink 둘 다 됩니다! (추가 다운로드 필요 없음)
2.  **Output Topic:** 데이터를 받을 토픽을 미리 생성해둡니다.

```bash
    # 터미널에서 실행
    docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-topics.sh \
      --create --bootstrap-server localhost:9092 --topic output-topic
```

> 문법에 대해 궁금하다면 ? [[PyFlink + Kafka 연동 완벽 가이드 ⭐️]] 참고 

###  Python Code (`kafka_sink.py`)

```python
import os
from pyflink.common import SimpleStringSchema, WatermarkStrategy, SerializationSchema
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaSink, KafkaRecordSerializationSchema, KafkaOffsetsInitializer
from pyflink.common.typeinfo import Types

def run_kafka_sink():
    # 1. 환경 설정
    env = StreamExecutionEnvironment.get_execution_environment()
    
    # JAR 파일 로딩 (Source와 동일)
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)

    # 2. Source 설정 (읽기)
    source = KafkaSource.builder() \
        .set_bootstrap_servers("kafka:9092") \
        .set_topics("input-topic") \
        .set_group_id("sink-group") \
        .set_starting_offsets(KafkaOffsetsInitializer.latest()) \
        .set_value_only_deserializer(SimpleStringSchema()) \
        .build()

    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")

    # 3. Transformation (가공)
    # 들어온 문자열에 " - Processed" 를 붙임 & 단어마다 앞글자 대문자 
    processed_stream=stream.map(lambda x:f"Processed: {x.title()}", output_type=Types.STRING())

    # 4. Sink 설정 (쓰기) ⭐️ 핵심
    sink = KafkaSink.builder() \
        .set_bootstrap_servers("kafka:9092") \
        .set_record_serializer(
            KafkaRecordSerializationSchema.builder()
                .set_topic("output-topic")  # 데이터를 넣을 토픽
                .set_value_serialization_schema(SimpleStringSchema()) # 문자열로 변환해서 저장 
                .build()
        ) \
        .build()

    # 5. Sink 연결
    # sinkTo() 메서드를 사용하여 스트림의 끝을 지정 
    processed_stream.sink_to(sink)

    env.execute("Flink Kafka Sink Job")

if __name__ == '__main__':
    run_kafka_sink()
```

---
## 🧐 Deep Dive: `.sink_to(sink)` vs `.add_sink(...)`

코드의 마지막에 등장하는 이 메서드는 **스트림 파이프라인의 종착역(Terminal Operation)** 을 정의합니다.

```python
# 5. Sink 연결
processed_stream.sink_to(sink)
```

### 역할: "배관 연결"

- **Source**가 수도꼭지라면, **Stream**은 물이 흐르는 파이프이고, **Sink**는 배수구입니다.
- `sink_to(sink)`는 흐르고 있는 파이프(`processed_stream`)의 끝을 미리 만들어둔 배수구(`sink`)에 물리적으로 **"체결"** 하는 명령어입니다.
- 이 코드가 실행되는 순간, Flink의 **Job Graph(작업 지도)** 에 "데이터 내보내기"라는 마지막 노드가 그려집니다.

### 왜 `add_sink()`가 아니라 `sink_to()` 인가요? ⭐️ (중요)

구글링을 하다 보면 `stream.add_sink(...)`라는 코드를 더 많이 보게 될 겁니다. 이 둘의 차이를 아는 것이 "최신 Flink 개발자"의 증거입니다.

- `add_sink(...)` (구버전 API): Flink 1.14 이전 방식. `FlinkKafkaProducer` 등을 쓸 때 사용.
- `sink_to(...)` (신버전 API - Sink V2): Flink 1.15부터 도입된 **"Unified Sink API"**.
	- **장점:** 트랜잭션(Exactly-Once), 작은 파일 병합(Compaction) 등의 복잡한 기능을 Flink 내부에서 더 안정적으로 처리해 줍니다.