---
aliases:
  - IOT 프로젝트
  - PYFlink실전
  - 센서데이터 평균
tags:
  - PyFlink
  - Kafka
  - Python
related:
  - "[[PyFlink_Windows]]"
  - "[[Apache_Kafka_Concept]]"
  - "[[Python_Logging]]"
  - "[[Python_Sys_Module]]"
  - "[[Kafka_Python_Producer]]"
  - "[[PyFlink_KeyedProcessFunction]]"
linked:
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/iot_producer.py
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/iot_consumer.py
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/flink_iot_job.py
---
## Project Overview (개요)

**"IoT 센서에서 실시간으로 들어오는 온도의 10초 평균을 구하라!"**

- **시나리오:** 
- 수만 개의 IoT 센서가 1초마다 온도 데이터를 보냅니다. 
- 우리는 지역(State)별로 10초마다 평균 온도를 계산하여 대시보드(Consumer)로 보내야 합니다.

- **아키텍처 흐름:** 
1. **Sensor (Producer):** `iot-topic`으로 랜덤 온도 데이터 전송 (`ca`, `ny`, `az`).
2. **Flink (Processor):** 데이터를 받아 **10초 단위로 윈도우**를 씌우고 **평균(Avg)** 을 계산. 
3. **Dashboard (Consumer):** `iot-average-topic`에서 계산된 결과를 실시간 수신

---
## Step 1: Data Generator (Producer)

- Kafka로 데이터를 쏘는 역할입니다.
- **Topic:** `iot-topic`
-  **Format:** `{"state": "ca", "temperature": 25.5}`
- **특징:** 1초에 한 번씩 데이터를 보냅니다.

```python
import logging
import sys
import time
import json
import random
# Kafka 라이브러리 임포트
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable

# 1. 로깅 설정 (print보다 더 자세한 정보를 남기기 위함)
# 언제(asctime), 어떤 레벨(levelname)로 메시지가 나가는지 찍습니다.
logging.basicConfig(
    level=logging.INFO,
    stream=sys.stdout,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ---------------------------------------------------------
# 2. 연결 재시도 로직 (The Zombie Pattern 🧟)
# ---------------------------------------------------------
# Docker 환경에서는 Kafka 컨테이너가 켜지는 데 시간이 걸립니다.
# 파이썬이 먼저 켜져서 "Kafka 없는데?" 하고 죽는 것을 방지하기 위해 10번 시도합니다.
producer = None
for i in range(10):
    try:
        print(f"📡 Kafka 연결 시도 중... ({i+1}/10)")
        
        # Producer(발신자) 조립
        producer = KafkaProducer(
            bootstrap_servers='kafka:9092',  # Docker 내부 통신용 주소
            
            # [중요] 데이터 포장기 (Serializer)
            # 우리는 파이썬 딕셔너리({'state': 'ca'...})를 보낼 거지만,
            # Kafka는 바이트(Bytes)만 받습니다.
            # 그래서 딕셔너리 -> JSON 문자열 -> UTF-8 바이트로 변환해서 보냅니다.
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        print("✅ Kafka 연결 성공!")
        break # 연결 성공하면 반복문 탈출!
        
    except NoBrokersAvailable:
        # Kafka가 아직 안 켜졌으면 3초 기다렸다가 다시 시도
        print("⏳ Kafka 브로커가 아직 준비 안 됨. 3초 대기...")
        time.sleep(3)
else:
    # 10번(약 30초) 다 시도했는데도 실패하면 에러를 내고 프로그램 종료
    raise RuntimeError("❌ Kafka 브로커를 10회 재시도 후에도 연결할 수 없습니다.")

# ---------------------------------------------------------
# 3. 데이터 생성 및 전송 (Main Loop)
# ---------------------------------------------------------
topic = 'iot-topic' # 데이터를 쏘아 보낼 토픽 이름
print(f"Producing IoT messages to topic: {topic}")

# 시뮬레이션할 지역 목록 (California, New York, Arizona)
states = ["ca", "ny", "az"]

# 1000번 반복하면서 가짜 데이터 생성
for i in range(1000):
    # 1. 랜덤 데이터 생성 (Random Generator)
    state = random.choice(states)  # 지역 중 하나 뽑기
    
    # 10.0도 ~ 40.0도 사이의 실수(float)를 뽑고, 소수점 1자리에서 반올림
    temperature = round(random.uniform(10.0, 40.0), 1) 
    
    # 2. 보낼 메시지 포장 (Dictionary)
    message = {
        'state': state,
        'temperature': temperature
    }
    
    # 3. 우편함에 넣기 (Send)
    # value_serializer 덕분에 자동으로 JSON 바이트로 변환되어 날아감
    producer.send(topic, value=message)
    
    # 로그 남기기 (잘 갔나 확인용)
    logging.info(f"📤 Sent: {message}")
    
    # 1초 쉬기 (너무 빨리 보내면 정신없으니까)
    time.sleep(1)

# 4. 마무리 (Flush)
# 혹시 전송 안 되고 버퍼에 남아있는 데이터가 있으면 강제로 다 밀어내고 종료
producer.flush()
print("🎉 모든 데이터 전송 완료!")
```


---
## Step 2: Flink Job Logic (The Core)


가장 중요한 부분입니다
Kafka에서 데이터를 읽어서, JSON을 파싱하고, 10초 윈도우 집계를 수행한 뒤, 다시 Kafka로 쏘는 코드입니다.

```python
import logging
import sys
import json
import time

# PyFlink 필수 라이브러리 임포트
from pyflink.common import Types, WatermarkStrategy
from pyflink.common.serialization import SimpleStringSchema
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.functions import KeyedProcessFunction, RuntimeContext
from pyflink.datastream.state import ValueStateDescriptor
# Kafka 연결을 위한 커넥터 임포트
from pyflink.datastream.connectors.kafka import (
    KafkaSource, 
    KafkaSink, 
    KafkaRecordSerializationSchema, 
    KafkaOffsetsInitializer
)

# 1. 로깅 설정 (Docker 로그에서 확인하기 위함)
logging.basicConfig(
    level=logging.INFO,
    stream=sys.stdout,
    format="%(message)s"
)

# ---------------------------------------------------------
# 2. 핵심 로직: 10초 윈도우 평균 계산기 (Stateful Function)
# ---------------------------------------------------------
class AverageProcessFunction(KeyedProcessFunction):
    def __init__(self):
        # 상태(State) 변수들을 담을 껍데기만 선언
        self.sum_state = None
        self.count_state = None
        self.timer_state = None
        
    def open(self, runtime_context: RuntimeContext):
        """
        [초기화 단계]
        Flink가 실행될 때 딱 한 번 호출됩니다.
        여기서 '상태 저장소(State)'를 Flink에게 요청해서 배정받습니다.
        """
        # 1. 온도 합계를 저장할 금고 (Float)
        self.sum_state = runtime_context.get_state(
            ValueStateDescriptor("sum", Types.FLOAT())
        )
        # 2. 데이터 개수를 저장할 금고 (Long/Int)
        self.count_state = runtime_context.get_state(
            ValueStateDescriptor("count", Types.LONG())
        )
        # 3. 타이머 실행 시간을 기억할 금고 (Long) -> 중복 실행 방지용
        self.timer_state = runtime_context.get_state(
            ValueStateDescriptor("timer", Types.LONG())
        )
     
    def process_element(self, value, ctx: KeyedProcessFunction.Context):
        """
        [데이터 처리 단계]
        Kafka에서 데이터가 1건 들어올 때마다 실행됩니다.
        value: (지역, 온도) 튜플 형태
        """
        # 튜플에서 온도값(인덱스 1) 추출
        current_temp = value[1]
        
        # 1. 상태 저장소에서 현재까지의 합계와 개수 꺼내오기
        current_sum = self.sum_state.value()
        current_count = self.count_state.value()
        
        # 처음이라 값이 없으면(None) 0으로 초기화
        if current_sum is None:
            current_sum = 0.0
        if current_count is None:
            current_count = 0
            
        # 2. 이번에 들어온 온도 누적하기
        current_sum += current_temp
        current_count += 1
        
        # 3. 누적된 값을 다시 상태 저장소에 업데이트 (Update)
        self.sum_state.update(current_sum)
        self.count_state.update(current_count)
        
        # 4. 타이머 설정 (알람 맞추기)
        current_timer = self.timer_state.value()
        
        # 아직 타이머가 안 맞춰져 있다면 (첫 데이터라면)
        if current_timer is None:
            # 현재 서버 시간 + 10초 (10000ms) 뒤로 알람 설정
            timer_time = ctx.timer_service().current_processing_time() + 10000
            
            # Flink 시스템에 알람 등록
            ctx.timer_service().register_processing_time_timer(timer_time)
            
            # "나 알람 맞췄어"라고 상태에 기록 (중복 방지)
            self.timer_state.update(timer_time)
    
    def on_timer(self, timestamp: int, ctx: KeyedProcessFunction.OnTimerContext):
        """
        [알람 울림 단계]
        설정해둔 10초가 지나면 Flink가 이 함수를 깨웁니다.
        """
        # 저장해둔 합계와 개수 꺼내기
        current_sum = self.sum_state.value()
        current_count = self.count_state.value()
        
        # 데이터가 하나라도 있을 때만 계산
        if current_count is not None and current_count > 0:
            # 평균 계산
            avg_temp = current_sum / current_count
            
            # 현재 처리 중인 지역명(Key) 가져오기
            state_key = ctx.get_current_key()
            
            # 결과 메시지 생성
            result_msg = f"State:{state_key}, 10초 평균 온도 :{avg_temp:.2f}°C"
            
            # 결과 내보내기 (Yield를 쓰면 Downstream으로 전달됨)
            yield result_msg
            
            # 로그에도 찍어보기
            print(f"출력!:{result_msg}")
            
        # [중요] 윈도우가 끝났으니 상태를 초기화 (설거지)
        # 이걸 안 하면 다음 10초 평균에 이전 값이 섞입니다.
        self.sum_state.clear()
        self.count_state.clear()
        self.timer_state.clear()

# ---------------------------------------------------------
# 3. 메인 파이프라인 구성
# ---------------------------------------------------------
def run_job():
    # Flink 실행 환경 설정
    env = StreamExecutionEnvironment.get_execution_environment()
    env.set_parallelism(1) # 테스트니까 1개만 띄움
    
    # Kafka 커넥터 JAR 파일 로드 (Docker 경로 기준)
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)
    
    # Source: Kafka에서 데이터 읽기 설정
    source = (
        KafkaSource
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_topics("iot-topic")            # 읽어올 토픽
        .set_group_id("flink-iot-group")    # 컨슈머 그룹 ID
        .set_starting_offsets(KafkaOffsetsInitializer.latest()) # 최신 데이터부터
        .set_value_only_deserializer(SimpleStringSchema())      # 문자열로 읽기
        .build()
    )
    
    # 데이터 스트림 생성
    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")
    
    # 파이프라인 연결 (Chaining)
    processed_stream = (
        stream
        # 1. JSON 문자열 -> 파이썬 딕셔너리 변환
        .map(lambda x: json.loads(x), output_type=Types.PICKLED_BYTE_ARRAY())
        
        # 2. 딕셔너리 -> (지역, 온도) 튜플 변환
        # KeyBy를 쓰기 위해 (Key, Value) 형태로 만듦
        .map(lambda obj: (obj['state'], float(obj['temperature'])), 
             output_type=Types.TUPLE([Types.STRING(), Types.FLOAT()]))
        
        # 3. 지역(State)별로 그룹핑 (Partitioning)
        .key_by(lambda x: x[0])
        
        # 4. 우리가 만든 핵심 로직(AverageProcessFunction) 적용
        # 최종 결과는 String 형태로 나감
        .process(AverageProcessFunction(), output_type=Types.STRING())
    )
    
    # Sink: 처리된 결과를 다시 Kafka로 내보내기 설정
    sink = (
        KafkaSink
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_record_serializer(
            KafkaRecordSerializationSchema
            .builder()
            .set_topic("iot-average-topic")  # 결과를 저장할 토픽
            .set_value_serialization_schema(SimpleStringSchema())
            .build()
        )
        .build()
    )
    
    # 스트림 끝에 Sink 연결
    processed_stream.sink_to(sink)
    
    print("Flink IoT Job started...")
    # Job 실행 시작
    env.execute("PyFlink IoT Job")
    
if __name__ == "__main__":
    run_job()
```

---
## Step 3: Dashboard (Consumer)

결과를 확인하는 모니터링 코드입니다. 
Flink가 계산해서 `iot-average-topic`에 넣은 데이터를 읽습니다. 

- **Topic:** `iot-average-topic`
- **Action:** 10초마다 평균값이 잘 찍히는지 확인.

```python
import logging
import time
import sys
# Kafka 라이브러리 임포트
from kafka import KafkaConsumer
from kafka.errors import NoBrokersAvailable

# 1. 로깅 설정 (print보다 더 자세한 정보를 남기기 위함)
# 언제(asctime), 어떤 레벨(levelname)로 메시지가 왔는지 표준 출력(stdout)으로 찍습니다.
logging.basicConfig(
    level=logging.INFO,
    stream=sys.stdout,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ---------------------------------------------------------
# 2. 연결 재시도 로직 (The Zombie Pattern 🧟)
# ---------------------------------------------------------
# Docker 환경에서는 Kafka 컨테이너가 켜지는 데 시간이 걸립니다.
# 파이썬이 먼저 켜져서 "Kafka 없는데?" 하고 죽는 것을 방지하기 위해 10번 시도합니다.
consumer = None
for i in range(10):
    try:
        print(f"📡 Kafka 연결 시도 중... ({i+1}/10)")
        
        # Consumer(수신기) 조립
        consumer = KafkaConsumer(
            'iot-average-topic',             # 구독할 토픽 (Flink가 결과를 쏘는 곳과 이름이 같아야 함!)
            bootstrap_servers='kafka:9092',  # Docker 내부 통신용 주소
            
            # [중요] 처음 켰을 때, 쌓여있는 옛날 데이터는 무시하고 '지금부터' 들어오는 것만 봄
            # 테스트할 때 실시간성을 확인하기 좋습니다.
            auto_offset_reset='latest',      
            
            # 읽었다는 표시(Offset Commit)를 자동으로 함 (편의성)
            enable_auto_commit=True,         
            
            # 컨슈머 그룹 ID: 나의 소속 팀 (이게 있어야 어디까지 읽었는지 기억함)
            group_id='iot-consumer-group',   
            
            # [중요] 데이터 해독기
            # Flink가 String으로 쏘기 때문에, 바이트를 UTF-8 문자열로 디코딩해야 함
            value_deserializer=lambda x: x.decode('utf-8')
        )
        print("✅ Kafka 연결 성공!")
        break # 연결 성공하면 반복문 탈출!
        
    except NoBrokersAvailable:
        # Kafka가 아직 안 켜졌으면 3초 기다렸다가 다시 시도
        print("⏳ Kafka 브로커가 아직 준비 안 됨. 3초 대기...")
        time.sleep(3)
else:
    # 10번(약 30초) 다 시도했는데도 실패하면 에러를 내고 프로그램 종료
    raise RuntimeError("❌ Kafka 브로커를 10회 재시도 후에도 연결할 수 없습니다.")

print("🎧 Listening for messages on 'iot-average-topic'...")   

# ---------------------------------------------------------
# 3. 메시지 수신 무한 루프 (Main Loop)
# ---------------------------------------------------------
# consumer는 Generator라서 for문으로 돌리면 메시지가 올 때까지 대기(Block)합니다.
for message in consumer:
    # message.value에는 value_deserializer로 해독된 '문자열'이 들어있습니다.
    # 예: "State:ca, 10초 평균 온도 :25.50°C"
    logging.info(f"📩 Received: {message.value}")
```


---
## Execution Guide (실행 방법)

Docker 환경에서 **터미널 창 3개**를 열고 순서대로 실행하세요. (모든 파일은 `/opt/flink/playground/src/`에 있다고 가정)

### 터미널 1: Flink Job 실행 (계산기 켜기)

가장 먼저 Flink를 대기 상태로 만듭니다.

```bash
docker exec -it apache-flink-jobmanager-1 bash

# Kafka 커넥터 경로 주의
/opt/flink/bin/flink run \
  -py /opt/flink/playground/src/flink_iot_job.py \
  -j /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

### 주의사항! 

`python3 .. .py` 실행할때  kafka가 존재 하지 않는다는 에러가뜬다

```bash
# docker exec -it apache-flink-jobmanager-1 bash 에 들어가서 
pip3 install kafka-python 
```

### 터미널 2: Consumer 실행 (결과창 켜기)

결과를 볼 준비를 합니다. 아직은 데이터가 안 옵니다.

```bash
docker exec -it apache-flink-jobmanager-1 bash

python3 /opt/flink/playground/src/iot_consumer.py
```

### 터미널 3: Producer 실행 (데이터 쏘기!)

이제 데이터를 쏘기 시작

```bash
docker exec -it apache-flink-jobmanager-1 bash

python3 /opt/flink/playground/src/iot_producer.py
```

## Expected Result (예상 결과)

**Producer (터미널 3):** `Sent: {'state': 'ca', 'temperature': 30.1}` ... (1초마다 쫘라락 찍힘)

**Consumer (터미널 2):** 아무 반응 없다가 **10초 뒤에** 뭉텅이로 결과가 나왔다~!!!!!!!

```json
2026-02-12 11:43:32,857 - INFO - Received: State:ny, 10초 평균 온도 :28.47°C
2026-02-12 11:43:34,848 - INFO - Received: State:az, 10초 평균 온도 :20.41°C
2026-02-12 11:43:36,851 - INFO - Received: State:ca, 10초 평균 온도 :26.70°C
2026-02-12 11:43:46,850 - INFO - Received: State:ny, 10초 평균 온도 :22.63°C
```

---
### 결론: 어떻게 실행하면 되나요?

이제 터미널 3개 띄울 필요 없이, 아주 우아하게 실행할 수 있습니다.

**1. 실행 (데몬 모드)**

Bash

```
docker-compose up -d
```

_(센서, Flink, 대시보드 등 모든 컨테이너가 한 번에 실행됩니다.)_

**2. 로그 확인 (잘 돌아가는지 감시)**

Bash

```
# 센서(Producer)가 데이터 쏘는 거 보기
docker-compose logs -f iot-producer

# 대시보드(Consumer)가 데이터 받는 거 보기
docker-compose logs -f iot-consumer
```

이제 진짜 엔지니어처럼 시스템을 구축하신 겁니다! 축하드립니다! 🎉

```yml
services:
  kafka:
    image: apache/kafka:3.7.0
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      - KAFKA_NODE_ID=1
      - KAFKA_PROCESS_ROLES=broker,controller
      - KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
      - KAFKA_LISTENERS=INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093
      - KAFKA_ADVERTISED_LISTENERS=INTERNAL://kafka:9092,EXTERNAL://localhost:29092
      - KAFKA_INTER_BROKER_LISTENER_NAME=INTERNAL
      - KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_AUTO_CREATE_TOPICS_ENABLE=true
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1
      - KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1
      - KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1
    volumes:
      - kafka_data:/var/lib/kafka/data

  jobmanager:
    build: .
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
    volumes:
      - ./playground:/opt/flink/playground
      - ./lib/flink-sql-connector-kafka-3.1.0-1.18.jar:/opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar

  taskmanager:
    build: .
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2
    volumes:
      - ./playground:/opt/flink/playground
      - ./lib/flink-sql-connector-kafka-3.1.0-1.18.jar:/opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar

  # ----------------------------------------
  # [추가됨] IoT 센서 데이터 생성기 (Producer)
  # ----------------------------------------
  iot-producer:
    build: .
    depends_on:
      - kafka
    volumes:
      - ./playground:/opt/flink/playground
    # 1. 라이브러리 설치 -> 2. 스크립트 실행 (백그라운드) -> 3. 컨테이너 유지 (tail)
    command: >
      sh -c "pip install kafka-python && 
             nohup python /opt/flink/playground/src/producer.py > /proc/1/fd/1 2>/proc/1/fd/2 & 
             tail -f /dev/null"

  # ----------------------------------------
  # [추가됨] 대시보드 결과 확인용 (Consumer)
  # ----------------------------------------
  iot-consumer:
    build: .
    depends_on:
      - kafka
    volumes:
      - ./playground:/opt/flink/playground
    # 1. 라이브러리 설치 -> 2. 스크립트 실행 (백그라운드) -> 3. 컨테이너 유지 (tail)
    command: >
      sh -c "pip install kafka-python && 
             nohup python /opt/flink/playground/src/consumer.py > /proc/1/fd/1 2>/proc/1/fd/2 & 
             tail -f /dev/null"

volumes:
  kafka_data:
```

docker-compose up -d --build