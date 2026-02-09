---
aliases:
  - PyFlink Kafka Source
  - Flink Kafka
tags:
  - Flink
  - Kafka
related:
  - "[[Flink_Kafka_Docker_Setup]]"
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_코드 해부_common ⭐️]]"
---
# PyFlink: Kafka Source 연결

## 한줄 용약

**"실시간으로 쏟아지는 Kafka 토픽(`KafkaSource`)을 빨대 꽂아서 읽어오기."**

---
##  Why: 왜 이렇게 바꾸나?

* **현실성:** 실무에서는 데이터를 코드 안에 하드코딩(`"Alice", "Bob"`)하지 않습니다. 외부 시스템(Kafka)에서 계속 들어오는 데이터를 처리해야 합니다.
* **무한성:** Kafka는 끝이 없는 **Unbounded Stream**입니다.

---
## Practical Context (실무 활용)

* **로그 수집:** 웹서버 Access Log가 Kafka로 들어오면 Flink가 실시간으로 읽어서 에러를 탐지함.
* **주문 처리:** 고객의 주문 내역이 Kafka Topic에 쌓이면, Flink가 읽어서 재고를 차감함.

---
##  Prerequisites (필수 준비물) ️

PyFlink는 깡통입니다. Kafka와 대화하려면 **통역사(Jar 파일)** 가 필요합니다.
* **필요한 Jar:** `flink-sql-connector-kafka-3.0.0-1.18.jar` (버전에 맞춰 다운로드 필요)
* **다운로드:** [Maven Central](https://mvnrepository.com/artifact/org.apache.flink/flink-sql-connector-kafka)

> Dockerfile에서 이미 `/opt/flink/lib`에 JAR 파일을 넣어뒀으므로, 매번 다운로드하거나 경로를 지정할 필요가 없음.

---
## Code Core Points

1.  **`KafkaSource.builder()`**: 카프카 접속 정보(IP, Port, Topic) 설정.
2.  **`set_starting_offsets`**: 어디서부터 읽을지 결정 (보통 `earliest`로 테스트).
3.  **`set_value_only_deserializer`**: 메시지의 내용(Value)만 문자열로 가져오기.
4.  **`env.add_jars()`**: **(핵심)** Kafka 커넥터 Jar 파일 로드.

## 6. PyFlink Code (kafka_source_example.py)

```python
import os
from pyflink.common import SimpleStringSchema, WatermarkStrategy
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaOffsetsInitializer

def run_kafka_source():
    # 1. 실행 환경 설정 (Stream 모드가 기본)
    env = StreamExecutionEnvironment.get_execution_environment()
    
    # Dockerfile에서 넣어둔 JAR 파일 경로 사용
    # 1. JAR 파일 로딩 (flink run -j 옵션 쓰면 생략 가능하지만 명시적으로 둠)
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.0.0-1.18.jar"
    # 환경에 JAR 추가 (이게 있어야 Kafka 클래스를 찾음)
    env.add_jars(jar_path)
    
    print(f"✅ Loaded JAR from: {jar_path}")

    # 2. Kafka Source 설정 (빌더 패턴)
    # (주의: Docker 내부에서 실행하므로 localhost 대신 'host.docker.internal'이나 'kafka' 서비스명 사용 권장)
    source = KafkaSource.builder() \
        .set_bootstrap_servers("kafka:9092") \
        .set_topics("input_topic") \
        .set_group_id("my-flink-group") \
        .set_starting_offsets(KafkaOffsetsInitializer.earliest()) \
        .set_value_only_deserializer(SimpleStringSchema()) \
        .build()
        # earliest(): 처음부터 다 읽기 / latest(): 지금부터 들어오는 것만 읽기

    # 3. DataStream 생성 (Source 연결)
    # WatermarkStrategy.no_watermarks(): 이벤트 시간 처리 안 할 때 사용 (단순 조회용)
    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")

    # 4. 데이터 출력 (Sink)
    stream.print()

    # 5. 실행
    env.execute("PyFlink Kafka Source Example")

if __name__ == '__main__':
    run_kafka_source()
```

### ① `file:///opt/flink/lib/...`

- Dockerfile에서  `RUN curl -o ...` 명령어로 저장한 위치
- `env.add_jars()`를 쓸 때 앞에 `file://`을 붙여야 "이건 내 로컬(컨테이너) 파일 시스템에 있는 거야"라고 인식

### ② `set_bootstrap_servers("kafka:9092")`

- **중요:** Docker 컨테이너 안에서 실행되므로 `localhost`는 "Flink 컨테이너 자기 자신"을 의미합니다.
- Kafka가 다른 컨테이너에 있다면, **docker-compose 서비스 이름** (예: `kafka`)을 써야 통신이 된다.

### ③ `env.from_source(...) ​`

- **의미:** "자, 위에서 만든 소스(기계)를 Flink 환경(env)에 연결할게!"(FileSource를 environment에 등록)
- 전선을 콘센트에 꽂듯 `env`에 등록해야 합니다.

```python
1env.from_source(Source, WatermarkStrategy, SourceName)
```

- `Source` : Kafka source
- `WatermarkStrategy` : 스트림 데이터의 시간 기준을 어떻게 잡을지, 파일 읽기(배치)라면 무조건
- **`WatermarkStrategy.no_watermarks()`**: "이 데이터는 실시간이 아니라 이미 저장된 파일이야. 시간 순서 따질 필요 없어." (배치 처리할 때 씀, 단순조회용)
- `SourceName` : Flink 대시보드(웹)에서 보여질 이름.

---
##  How to Run (실행)

Docker 컨테이너에 접속해서 실행하면 `-j` 옵션 없이 아주 심플하게 실행됩니다.

> **상세 트러블슈팅 및 전체 가이드:** [[PyFlink + Kafka 연동 완벽 가이드]] 참고 




