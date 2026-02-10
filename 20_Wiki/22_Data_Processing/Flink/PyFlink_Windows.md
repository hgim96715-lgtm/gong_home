---
aliases:
  - PyFlink Windows
  - Window Assigners
  - Tumbling Window
  - Sliding Window
  - Session Window
tags:
  - PyFlink
related:
  - "[[PyFlink_KeyBy_DeepDive]]"
  - "[[PyFlink_Operators_Basic]]"
  - "[[PyFlink_Kafka_코드해부_Common ⭐️]]"
linked:
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/kafka_windows.py
---
#  PyFlink Windows: 무한한 데이터를 유한하게 나누기

> [!QUOTE] 핵심 요약
> **"Windows are at the heart of processing infinite streams."** 
> 끝이 없는 데이터 스트림을 **일정 기준(시간, 개수)** 으로 잘라서 **"유한한 버킷(Bucket)"** 에 담는 기술입니다.
> * "지난 1분간 주문 건수", "최근 3번의 클릭" 등을 구할 때 필수입니다.

---
## Why? 왜 필요한가요? 

스트림 데이터는 **끝이 없습니다(Infinite).**
* **문제:** "평균을 구해줘"라고 하면, 데이터가 끝날 때까지 기다려야 하는데 영원히 끝나지 않으니 계산을 시작조차 못 합니다.
* **해결:** "데이터 전체"가 아니라 **"지난 5분 동안"** 처럼 범위를 잘라줘야(Windowing) 집계(`sum`, `avg`)가 가능해집니다.

---
##  Window의 종류 (Window Assigners) 

 상황에 맞춰 골라 써야 합니다.

### ① Tumbling Window (텀블링) 

* **특징:** **크기가 고정**되어 있고, **겹치지 않습니다(Non-overlapping).** 
* **동작:** 12:00~12:05, 12:05~12:10 처럼 딱딱 끊어집니다. 데이터는 오직 하나의 윈도우에만 속합니다.
* **용도:** "매 5분마다 정산하기"

### ② Sliding Window (슬라이딩) 

* **특징:** **크기가 고정**되어 있지만, **겹칩니다(Overlap).** 
* **동작:** 윈도우 크기는 10분인데, 5분마다 결과를 냅니다. (`[12:00~12:10]`, `[12:05~12:15]`)
* **용도:** "최근 1시간 데이터를 10분마다 갱신해서 보여줘" (주식 이동평균선 등)

### ③ Session Window (세션) 

* **특징:** 크기가 **동적(Dynamic)** 입니다. **활동이 없으면(Inactivity)** 윈도우를 닫습니다. 
* **동작:** 사용자가 클릭을 막 하다가 5분간 아무것도 안 하면(Gap), 그때까지를 "하나의 세션"으로 묶습니다.
* **용도:** "유저의 1회 방문(Session) 동안 본 페이지 수"

---
## Practical Context: Kafka + PyFlink 변환 코드 

 **Tumbling Window (Processing Time)** 

```python
"""
[PyFlink] Word Count Window Example (Hybrid)
목표: Kafka에서 "hi" 같은 단어가 들어오면 30초 동안 모아서 카운트를 셉니다.
입력: "hi"
출력: "window result 30 seconds -> hi: 3"
"""

import os
from pyflink.common import SimpleStringSchema, WatermarkStrategy
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaSink, KafkaRecordSerializationSchema, KafkaOffsetsInitializer
from pyflink.datastream.window import TumblingProcessingTimeWindows
# Sliding 윈도우 임포트 (필요시 주석 해제)
#from pyflink.datastream.window import SlidingProcessingTimeWindows
from pyflink.common.time import Time

def run_window_wordCount():
    
    # 1. 환경 설정
    env = StreamExecutionEnvironment.get_execution_environment()
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)
    
    
    # 2. Source 설정
    source = (
        KafkaSource
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_topics("input-topic")
        .set_group_id("window-wordcount-group")
        .set_starting_offsets(KafkaOffsetsInitializer.latest())
        .set_value_only_deserializer(SimpleStringSchema())
        .build()
    )
    
    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")
    
    
    # 3. 데이터 변환 (Map)
    # 입력: "hi" -> 출력: ("hi", 1)
    mapped_stream = stream.map(
        lambda x: (x, 1),
        output_type=Types.TUPLE([Types.STRING(), Types.INT()])
    )
    
    # 4. KeyBy + Window + Reduce (핵심 로직)
    windowed_stream = (
        mapped_stream
        .key_by(lambda x: x[0])
        # [Option A] Tumbling Window (30초마다 한 번)
        .window(TumblingProcessingTimeWindows.of(Time.seconds(30)))
        
        # [Option B] Sliding Window (30초 창문을 10초마다 갱신) - 필요시 주석 해제하여 교체
        # .window(SlidingProcessingTimeWindows.of(Time.seconds(30), Time.seconds(10)))
        
        .reduce(lambda a, b: (a[0], a[1] + b[1]))
    )
    
    # 5. Sink 설정
    sink = (
        KafkaSink
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_record_serializer(
            KafkaRecordSerializationSchema
            .builder()
            .set_topic("output-topic")
            .set_value_serialization_schema(SimpleStringSchema())
            .build()
        )
    ).build()
    
    # 6. 결과 출력 포맷팅
    windowed_stream.map(
        lambda x: f"window result 30 seconds -> {x[0]}: {x[1]}",
        output_type=Types.STRING()
    ).sink_to(sink)
    
    env.execute("Kafka Window WordCount")

if __name__ == "__main__":
    run_window_wordCount()
```
---
## Detailed Analysis: 코드가 어떻게 도나요?

### Step 1. `key_by(...)`

- **이유:** 윈도우는 **Keyed Window**와 **Non-Keyed Window**로 나뉩니다.
	- **Keyed:** 병렬 처리 가능 (빠름). "Alice 윈도우"와 "Bob 윈도우"가 따로 돕니다.
	- **Non-Keyed:** 병렬 처리 불가 (싱글 스레드). `key_by` 없이 `.window_all()`을 쓰면 전체 데이터가 한 곳으로 몰려서 매우 느려집니다.

### Step 2. `.window(...)`

- **역할:** 흐르는 강물에 "칸막이"를 칩니다.
- **`TumblingProcessingTimeWindows.of(Time.seconds(10))`:**
    - 시스템 시간(Processing Time) 기준으로 10초 동안 들어온 데이터를 모읍니다.
    - 10초가 땡! 지나면 문을 닫고 다음 단계(`reduce`)로 보냅니다.

### Step 3. `.reduce(...)`

- **역할:** 10초 동안 모인 "Alice"들의 점수를 다 더합니다.
- **결과:** 10초에 한 번씩 `Window Result -> Alice: 50` 같은 메시지가 Kafka로 나갑니다. (실시간으로 하나씩 나오는 게 아님!)

