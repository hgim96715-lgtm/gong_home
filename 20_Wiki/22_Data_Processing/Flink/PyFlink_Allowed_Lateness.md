---
aliases:
  - Allowed Lateness
  - 지각생구제
  - Late Data
  - 허용된지연
tags:
  - PyFlink
related:
  - "[[PyFlink_Trigger_Watermark]]"
  - "[[PyFlink_Windows]]"
---
# PyFlink Allowed Lateness: "지각생, 봐준다!"

## Concept Summary (개념 한 줄 요약)

**Allowed Lateness**는 윈도우가 닫힌(종료 시간 + 워터마크 통과) 후에도, **설정된 시간만큼은 문을 잠그지 않고 기다려주는 '유예 기간(Grace Period)'** 기능입니다.

* **기본 동작:** 워터마크가 윈도우 종료 시간을 지나면, 윈도우는 즉시 닫히고 그 이후 도착한 데이터는 **버려집니다(Dropped)**.
* **Allowed Lateness:** "5초만 더 기다려줄게!"라고 설정하면, 늦게 온 데이터가 도착했을 때 **이미 닫힌 윈도우를 다시 열어서 결과를 갱신(Update)** 해줍니다.

---
## Why is it needed? (왜 필요한가?)

* **네트워크 지연 (Network Delay):** 현실 세계의 데이터 스트림은 네트워크 문제나 재시도(Retries) 때문에 **순서가 뒤죽박죽(Out-of-order)** 인 경우가 많습니다.
* **데이터 손실 방지:** 지각생 구제 장치가 없으면, 네트워크 렉으로 조금 늦게 도착한 소중한 데이터들이 전부 증발해버릴 위험이 있습니다.

---
## Practical Context (동작 시나리오) 

**상황:** `[00초 ~ 10초]` 윈도우 / **Allowed Lateness: 5초** 

1.  **Time 10s (워터마크 통과):**
    * 윈도우가 1차적으로 닫힙니다.
    * **결과 출력 (1차):** "0~10초 사이 합계는 50"
    * ⚠️ **중요:** 하지만 윈도우(State)를 삭제(`PURGE`)하지 않고 **유지**합니다.

2.  **Time 12s (지각생 도착):**
    * 데이터(시간: 09초)가 뒤늦게 도착했습니다! (유예 기간 15초 이내) 
    * Flink가 닫힌 윈도우를 **다시 엽니다(Re-open).** 
    * 기존 합계 50에 9초 데이터를 더해 다시 계산합니다.
    * **결과 출력 (수정본):** "0~10초 사이 합계는 **55**" (Trigger: `FIRE`)

3.  **Time 15s (유예 기간 종료):**
    * 이제 진짜 끝입니다. 윈도우를 메모리에서 완전히 삭제(`Cleanup`)합니다.
    * 이후에 들어오는 09초 데이터는 **버려집니다.**

---
## Code Core Points

* **위치:** `.window(...)` 바로 뒤에 붙입니다.
* **핵심 함수:** `{python}.allowed_lateness(Time.seconds(N))`
* **주의:** 지각생이 올 때마다 결과가 **여러 번(Update)** 출력되므로, 결과를 받는 쪽(Sink)에서 "덮어쓰기" 처리가 가능한지 확인해야 합니다.

---
##  Detailed Analysis (PyFlink + Kafka Code)

```python
import os
from pyflink.common import WatermarkStrategy, Time, SimpleStringSchema
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaOffsetsInitializer
from pyflink.datastream.window import TumblingEventTimeWindows
from pyflink.datastream.functions import ProcessWindowFunction

# 결과 출력용 함수 (지각생이 오면 여러 번 호출됨)
class ResultFunction(ProcessWindowFunction):
    def process(self, key, context, elements):
        data_list = [e for e in elements]
        count = len(data_list)
        # 윈도우 시간도 같이 출력해서 지각생 반영 여부 확인
        window_start = context.window().start
        window_end = context.window().end
        yield f"Window[{window_start}~{window_end}] Key:{key} Count:{count} (Data: {data_list})"

def run_allowed_lateness():
    env = StreamExecutionEnvironment.get_execution_environment()
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)

    # 1. Source: Kafka 연결
    source = KafkaSource.builder() \
        .set_bootstrap_servers("kafka:9092") \
        .set_topics("late-input") \
        .set_group_id("late-group") \
        .set_starting_offsets(KafkaOffsetsInitializer.latest()) \
        .set_value_only_deserializer(SimpleStringSchema()) \
        .build()

    # 2. Watermark Strategy (중요!)
    # 데이터 속에 있는 '진짜 시간'을 꺼내고, 
    # 3초 정도의 난류(Out-of-order)는 기본으로 인정해줌 (Watermark Delay)
    watermark_strategy = WatermarkStrategy \
        .for_bounded_out_of_orderness(Time.seconds(3)) \
        .with_timestamp_assigner(...) # 타임스탬프 추출 로직 필요

    stream = env.from_source(source, watermark_strategy, "Kafka Source")

    # 3. Window & Allowed Lateness 설정
    stream \
        .key_by(lambda x: x[0]) \
        .window(TumblingEventTimeWindows.of(Time.seconds(10))) \
        \
        # ⭐️ [핵심] 지각생 구제 기간 5초 설정
        # 윈도우 종료 후 5초까지는 데이터를 받아줌 
        .allowed_lateness(Time.seconds(5)) \
        \
        .process(ResultFunction()) \
        .print()

    env.execute("Allowed Lateness Example")
```

---
## Common Beginner Misconceptions (초보자 실수)

- **Q: "Allowed Lateness를 쓰면 결과가 늦게 나오나요?"**
    - **아니요!** 정시(Watermark 통과 시)에 **일단 1차 결과**가 나옵니다. 
    - 그리고 지각생이 오면 **추가로 수정된 결과**가 나옵니다. 기다리는 게 아니라 **"A/S(수정)"** 해주는 개념입니다.
        
- **Q: "Watermark Delay랑 뭐가 달라요?"**
    - `for_bounded_out_of_orderness(3초)`: 아예 3초 동안은 **윈도우를 닫지 않고 기다립니다.** (결과가 3초 늦게 나옴, 1번만 출력)
    - `allowed_lateness(5초)`: 윈도우는 제때 닫고 결과도 냅니다. 단지 **나중에 수정할 기회**를 5초 더 주는 겁니다. (결과가 여러 번 나옴)

- **Q: "유예 기간도 지나면요?"**
    - 그때는 진짜 버려집니다. 
    - 만약 이것조차 살리고 싶다면 **Side Output** (`.side_output_late_data(...)`) 기능을 써서 따로 빼야 합니다.