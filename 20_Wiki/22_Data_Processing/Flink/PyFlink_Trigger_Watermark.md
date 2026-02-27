---
aliases:
  - Trigger
  - Watermark
  - 윈도우발화
  - 트리거
tags:
  - PyFlink
related:
  - "[[SQL_Window_Functions]]"
  - "[[PyFlink_Windows]]"
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_Kafka_코드해부_Common ⭐️]]"
  - "[[PyFlink_SQL_Watermark]]"
---
# PyFlink Trigger & Watermark: 윈도우가 터지는 순간 결정하기

> [!QUOTE] **Concept Summary**
> **Trigger(트리거)**는 윈도우라는 양동이에 담긴 데이터를 **"언제 계산해서 내보낼지(Emit)"** 결정하는 결정권자입니다. 
> * "시간이 다 됐으니 내보내자!" (기본)
> * "데이터가 10개 찼으니 미리 내보내자!" (커스텀)

---
## Why is it needed? (왜 써야 하나요?)

모든 윈도우는 **기본 트리거(Default Trigger)** 를 가지고 있습니다. 
* **Event Time Window:** 워터마크(Watermark)가 윈도우 종료 시간을 지날 때 발화합니다. 
* **Processing Time Window:** 서버 시간이 윈도우 종료 시간에 도달했을 때 발화합니다. 

**하지만 이런 문제가 생길 수 있어요:**
1.  **지연 시간 문제:** 1시간짜리 윈도우를 설정하면, 1시간을 꼬박 기다려야 첫 결과가 나옵니다. "5분마다 중간 결과라도 보고 싶은데?" 
2.  **개수 기반 조건:** "시간은 안 됐지만 데이터가 100개 쌓이면 일단 내보내고 싶어!" 


---
##  Watermark: "기다림의 미학" 

트리거와 떼려야 뗄 수 없는 개념이 **Watermark(워터마크)**  입니다. 

* **정의:** "이제 이 시간($t$)보다 이전 데이터를 가진 이벤트는 더 이상 오지 않을 거야"라고 선언하는 타임스탬프입니다. 
* **역할:** 네트워크 지연 등으로 데이터가 뒤죽박죽(Out-of-order) 들어올 때, 윈도우를 언제 닫을지 정하는 기준이 됩니다.

---
## `CountTrigger` 사용법 & 문법 가이드

"데이터가 n개 모일 때마다 발사!"하는 트리거의 정확한 사용법입니다.

### ① Import (불러오기)

`window` 모듈 안에 들어있습니다. `Tumbling...` 윈도우랑 같은 곳에 산다고 생각하면 됩니다.

```python
from pyflink.datastream.window import CountTrigger
# 또는 한 번에
# from pyflink.datastream.window import TumblingProcessingTimeWindows, CountTrigger
```

### ② Syntax (문법 구조)

반드시 **`.window(...)` 바로 뒤**에 붙여야 합니다.

```python
stream
    .key_by(...)
    .window(...)  # 1. 윈도우를 먼저 만들고
    .trigger(CountTrigger.of(2)) # 2. 그 윈도우에 트리거를 장착!
```

- **함수:** `CountTrigger.of(n)`
- **파라미터 `n` (long):** 방아쇠를 당길 기준 개수. (2를 넣으면 2개마다 발사)

### ③ 동작 원리 (Behavior) 

`CountTrigger.of(2)`를 설정했을 때, 데이터 흐름에 따른 반응입니다.

|**들어온 데이터**|**현재 개수**|**트리거 반응**|**결과 출력**|**비고**|
|---|---|---|---|---|
|`Alice,1`|1개|`CONTINUE`|(없음)|2개가 안 돼서 대기|
|`Alice,1`|**2개**|**`FIRE`**|**`Alice: 2`**|**조건 달성! 발사! 🔫**|
|`Alice,1`|3개|`FIRE`*|`Alice: 3`|(주의) 기본적으로 계속 FIRE 함|

**⚠️ 주의 (Advanced):** `CountTrigger`는 발사 후 **개수를 0으로 초기화하지 않습니다.** (누적됨) 
만약 2개마다 **초기화해서** ("2", "2", "2"...) 보고 싶다면 `PurgingTrigger.of(CountTrigger.of(2))`라는 복합 트리거를 써야 합니다. 

---
##  Practical Context: 10초 윈도우, 하지만 2개마다 출력하기 🛠️

**"10초 윈도우 + CountTrigger(2)"** 조합을 구현해 봅시다. 
이 설정은 **"윈도우는 10초 동안 유지되지만, 데이터가 2개 쌓일 때마다 중간 결과를 계속 뱉어라"** 라는 뜻입니다. 

### 💻 PyFlink + Kafka 실전 예제

```python
"""
[PyFlink] Trigger Example
- 윈도우: 10초 Tumbling Window
- 트리거: 데이터 2개 도착할 때마다 결과 출력
- 입력 예시: "Alice-1"  (하이픈 주의!)
"""

from pyflink.common import WatermarkStrategy, SimpleStringSchema
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import (
    KafkaSource,
    KafkaOffsetsInitializer
)
from pyflink.datastream.window import (
    TumblingProcessingTimeWindows,
    CountTrigger
)
from pyflink.common.time import Time


def run_trigger_example():
    # 1️⃣ 실행 환경 설정
    env = StreamExecutionEnvironment.get_execution_environment()
    env.add_jars("file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar")

    # 2️⃣ Kafka Source 설정
    source = (
        KafkaSource.builder()
        .set_bootstrap_servers("kafka:9092")
        .set_topics("input-topic")
        .set_group_id("trigger-group")
        .set_starting_offsets(KafkaOffsetsInitializer.latest())
        .set_value_only_deserializer(SimpleStringSchema())
        .build()
    )

    stream = env.from_source(
        source,
        WatermarkStrategy.no_watermarks(),
        "Kafka Source"
    )

    # 3️⃣ 핵심 로직
    result_stream = (
        stream
        # "Alice-1" → ("Alice", 1)
        .map(
            lambda x: (x.split("-")[0], int(x.split("-")[1])),
            output_type=Types.TUPLE([Types.STRING(), Types.INT()])
        )

        # key 기준 묶기
        .key_by(lambda x: x[0])

        # 10초짜리 Processing Time Window
        .window(TumblingProcessingTimeWindows.of(Time.seconds(10)))

        # ⭐️ 데이터 2개마다 트리거 발사
        # ⚠️ 이걸 쓰면 "10초 끝나면 자동 발사"는 사라짐
        .trigger(CountTrigger.of(2))

        # 누적 합계
        .reduce(lambda a, b: (a[0], a[1] + b[1]))
    )

    # 4️⃣ 결과 출력
    result_stream.print()

    env.execute("Kafka Trigger Example")


if __name__ == "__main__":
    run_trigger_example()

```

### ⚠️ 주의사항 (Trigger의 덮어쓰기)

- **Q: 10초가 지났는데 데이터가 1개밖에 안 왔어요. 출력되나요?**
- **A: 아니요! 출력되지 않습니다.** 🙅‍♂️
    - `.trigger(CountTrigger)`를 설정하면 **기본 시간 트리거(Default Time Trigger)가 비활성화**됩니다.
    - 즉, Flink는 오직 "개수가 2개인가?"만 감시합니다. 10초가 지나도, 100초가 지나도 2개가 안 되면 절대 내보내지 않습니다.

---
## `on_element` 문법 완전 해부

```python
def on_element(self, element, timestamp, window, ctx):
```

- 이 함수는 **"데이터가 윈도우에 들어올 때마다"** 자동으로 실행됩니다.

1. `element` (주인공 데이터)
	-  **Role:** 방금 들어온 **실제 데이터**입니다.
2. `timestamp` (도착 시간)
	- **Role:** 이 데이터가 생성된(또는 도착한) **시간**입니다. / **용도:** "이 데이터가 늦게 왔나?" 체크할 때 씁니다.
3. `window` (소속 양동이)
	- **Role:** 이 데이터가 담길 **윈도우 객체**입니다.
	- **용도:** `window.max_timestamp()`를 통해 "이 윈도우는 언제 닫히지?"(종료 시간)를 알 수 있습니다.
4. `ctx` (만능 도구함 / Context)
	- **Role:** 트리거가 Flink 시스템에게 **명령을 내릴 수 있는 도구(Context)** 입니다.
	- **기능:**
		- `ctx.register_processing_time_timer(...)`: "나중에 알람 울려줘!" (타이머 등록)    
		- `ctx.get_partitioned_state(...)`: "내 사물함(State) 가져와!" (상태 접근)
		- `ctx.delete_processing_time_timer(...)`: "알람 취소해!"


### [심화] State(상태) 다루는 3단계 공식

`ctx` 도구함에서 꺼낸 **사물함(State)** 을 다룰 때는 무조건 이 순서를 지켜야 합니다.

```python
# 1. 사물함 열기 (Retrieve)
# ctx를 통해 미리 신청해둔(desc) 사물함 객체를 가져옵니다.
count_state = ctx.get_partitioned_state(self.count_state_desc)

# 2. 내용물 확인하기 (Access)
# .value()를 써야만 실제 값(Integer)이 나옵니다.
current_count = count_state.value()

# 3. 빈 깡통 체크 (Null Check) ⭐️ 중요!
# 처음 열었을 땐 0이 아니라 None입니다. 초기화 필수!
if current_count is None:
    current_count = 0

# 4. 값 갱신하기 (Update)
# 계산 후 다시 사물함에 넣습니다.
current_count+=1
count_state.update(current_count)
```

>**💡 핵심 요약:**
> `ctx`는 사물함 열쇠를 주는 역할이고, 실제 값을 꺼내거나(`value`) 넣을 땐(`update`) **State 객체**를 씁니다.

---
## TriggerResult 4가지 신호등 (The Signals)

- `from pyflink.datastream.window import TriggerResult` 로 불러와서 씁니다.

| **신호 (Enum)**        | **의미 (Command)** | **비유 (Analogy)**                  | **데이터 삭제 여부** |
| -------------------- | ---------------- | --------------------------------- | ------------- |
| **`CONTINUE`**       | **대기**           | "아직 멀었어. 일단 쌓아둬." (🔴 빨간불)        | X (유지)        |
| **`FIRE`**           | **발사 (출력만)**     | "중간 정산서 출력해! 근데 장부는 냅둬." (🟡 노란불) | X (유지)        |
| **`PURGE`**          | **삭제 (출력 안 함)**  | "에이 망쳤다. 싹 다 지워버려!" (🗑 휴지통)      | **O (삭제)**    |
| **`FIRE_AND_PURGE`** | **발사 + 삭제**      | "최종 정산서 뽑고, 장부 태워버려!" (🟢 초록불)    | **O (삭제)**    |

>**`FIRE` (발사)**:
>결과를 내보내지만, 윈도우 안의 데이터(`Alice, 100`)는 **그대로 남아있습니다** 또 `FIRE` 하면 **"이전 데이터 + 새 데이터"** 가 합쳐져서 나옵니다. (누적 계산)
>**`FIRE_AND_PURGE` (발사 후 삭제)**:
>결과를 내보내고, 윈도우 안을 **깨끗하게 비웁니다.** **다음번엔**0부터 다시 시작**합니다.

- **위치:** 커스텀 트리거 클래스(`Trigger` 상속)의 메서드 내부 (`on_element`, `on_processing_time` 등).
- **역할:** 함수의 **반환값(Return Value)** 으로 사용. Flink 엔진에게 내리는 **최종 명령어**

```python
def on_element(...): 
	if (조건):
		 return TriggerResult.FIRE # 명령 1: 발사! 
	 else: 
		 return TriggerResult.CONTINUE # 명령 2: 대기!
```

---
## 그 외 생명주기 메서드 (The Supporting Cast) 

`on_element`가 주인공이라면, 이들은 무대를 정리하거나 알람을 듣는 조연들입니다.

### `on_processing_time` & `on_event_time` (알람 시계)

`ctx.register_..._timer()`로 맞춰놓은 **알람이 울리면** 실행되는 함수입니다.

```python
# 서버 시간(Processing Time) 알람이 울리면 실행
def on_processing_time(self, time, window, ctx):
    # 예: "20초 지났으니 강제로 발사!"
    return TriggerResult.FIRE 

# 데이터 시간(Event Time/Watermark) 알람이 울리면 실행
def on_event_time(self, time, window, ctx):
    return TriggerResult.CONTINUE
```

- **`on_processing_time`**: 벽걸이 시계(서버 시간) 기준입니다. 데이터가 안 들어와도 시간이 흐르면 무조건 실행됩니다. (타임아웃 구현에 필수)
- **`on_event_time`**: 워터마크(데이터 속 시간) 기준입니다. `WatermarkStrategy`가 설정되어 있어야 작동합니다.

### `clear` (청소 반장)  ⭐️

윈도우가 수명을 다하고 **완전히 사라질 때(Purge)** 마지막으로 호출됩니다.

```python
def clear(self, window, ctx):
    # 내 사물함(State)을 찾아서
    count_state = ctx.get_partitioned_state(self.count_state_desc)
    # 깨끗하게 비운다! (메모리 누수 방지)
    count_state.clear()
```

- **Why?** `Trigger`는 계속 새로운 윈도우를 만듭니다. 다 쓴 윈도우의 `State`를 지우지 않으면, 메모리에 쓰레기가 계속 쌓여서 결국 **서버가 터집니다(OOM).**
- **Role:** 반드시 `state.clear()`를 호출해서 자원을 반납해야 합니다.
- `ctx.get_partitioned_state(...)`: "내 사물함(State) 가져와!" (상태 접근)


### `on_merge` (합체 로봇)

**세션 윈도우(Session Window)** 처럼 윈도우 크기가 유동적일 때만 쓰입니다.

```python
def on_merge(self, window, ctx):
    # 윈도우 A와 윈도우 B가 합쳐질 때, 상태값도 합쳐야 함
    pass
```

- **Role:** 두 개의 윈도우가 하나로 합쳐질 때, 각각의 사물함(State)에 있던 내용물을 하나로 합치는 로직을 넣습니다.
- _참고:_ `Tumbling`(고정 크기)이나 `Sliding` 윈도우에서는 합쳐질 일이 없으므로 보통 `pass` 하거나 비워둡니다.


---
## [심화] Time OR Count Trigger (하이브리드)

이 코드는 두 가지 조건을 모두 감시합니다.

1. **Count:** 데이터가 들어올 때마다 개수를 세서 N개가 되면 `FIRE`.
2. **Time:** 데이터가 들어오면 **"윈도우 끝나는 시간"** 에 알람(Timer)을 맞춰놓고, 시간이 되면 `FIRE`.

```python
"""
[PyFlink] Custom Trigger Example Job"
목표: 
  1. 데이터가 2개 모이면 즉시 발사 (Early Fire)
  2. 2개가 안 모여도 20초(윈도우 종료)가 되면 무조건 발사 (Timeout)
입력: "Alice,1"
"""

import os
from pyflink.common import WatermarkStrategy, SimpleStringSchema
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaSink, KafkaRecordSerializationSchema, KafkaOffsetsInitializer
from pyflink.datastream.window import TumblingProcessingTimeWindows, Trigger, TriggerResult
from pyflink.datastream.state import ValueStateDescriptor
from pyflink.common.time import Time

# ---------------------------------------------------------
# ⭐️ [심화] 커스텀 트리거 클래스 정의
# "시간(Time)과 개수(Count)를 동시에 감시하는 감시자"
# ---------------------------------------------------------
class CountTimeoutTrigger(Trigger):
    def __init__(self, max_count):
        self.max_count = max_count
        # 개수를 세기 위한 상태 저장소(State) 정의
        self.count_state_desc = ValueStateDescriptor('count', Types.INT())

    # 1. 데이터가 들어올 때마다 호출됨
    def on_element(self, element, timestamp, window, ctx):
        # (A) "시간 규칙" 복구: 윈도우 끝나는 시간에 알람을 등록함! ⏰
        # 이걸 안 하면 시간이 지나도 안 꺼짐.
        ctx.register_processing_time_timer(window.max_timestamp())

        # (B) "개수 규칙" 체크: 개수 세기 🔢
        count_state = ctx.get_partitioned_state(self.count_state_desc)
        current_count = count_state.value()
        if current_count is None:
            current_count = 0
        
        current_count += 1
        count_state.update(current_count)

        # 개수가 꽉 찼니?
        # 목표 개수(max)를 채웠는지 확인합니다.
        if current_count >= self.max_count:
        # [중요] 카운터를 0으로(Empty) 만듭니다.
        # 이걸 안 하면 다음 데이터부터는 조건이 계속 True가 되어 무한 발사됩니다.
            count_state.clear() # 개수 초기화
            # 윈도우 결과를 내보냅니다. (데이터는 유지됨)
            return TriggerResult.FIRE # 발사! 🔫
        # 아직 목표를 못 채웠으면 그냥 넘어갑니다.
        return TriggerResult.CONTINUE

    # 2. 시간이 다 됐을 때 호출됨 (알람 울림)
    def on_processing_time(self, time, window, ctx):
        # 윈도우 시간이 끝났으니 무조건 발사!
        return TriggerResult.FIRE

    # 3. 이벤트 타임 (여기선 안 씀)
    def on_event_time(self, time, window, ctx):
        return TriggerResult.CONTINUE
    
    # 4. 초기화 (윈도우 삭제 시 상태 청소)
    def clear(self, window, ctx):
        count_state = ctx.get_partitioned_state(self.count_state_desc)
        count_state.clear()

    # 5. 병합 (세션 윈도우용 - 여기선 단순 합치기)
    def on_merge(self, window, ctx):
        pass

# ---------------------------------------------------------
# 메인 파이프라인
# ---------------------------------------------------------
def run_custom_trigger():
    env = StreamExecutionEnvironment.get_execution_environment()
    # JAR 파일 경로 (환경에 맞게 수정)
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)
    
    # Source
    source = KafkaSource.builder() \
        .set_bootstrap_servers("kafka:9092") \
        .set_topics("input-topic") \
        .set_group_id("custom-trigger-group") \
        .set_starting_offsets(KafkaOffsetsInitializer.latest()) \
        .set_value_only_deserializer(SimpleStringSchema()) \
        .build()
    
    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")
    
    result_stream = (
        stream
        .map(lambda x: (x.split(",")[0], int(x.split(",")[1])), 
             output_type=Types.TUPLE([Types.STRING(), Types.INT()]))
        .key_by(lambda x: x[0])
        
        # 20초 윈도우 설정
        .window(TumblingProcessingTimeWindows.of(Time.seconds(20)))
        
        # ⭐️ 우리가 만든 커스텀 트리거 장착! (2개 OR 20초)
        .trigger(CountTimeoutTrigger(max_count=2))
        
        .reduce(lambda a, b: (a[0], a[1] + b[1]))
    )
    
    result_stream.print()
   env.execute("Custom Trigger Example Job")

if __name__ == "__main__":
    run_custom_trigger()
```


----

- **`ctx.register_processing_time_timer(window.max_timestamp())`**
    - 이 한 줄이 바로 **"사라졌던 시간 규칙을 되살리는 마법"** 입니다.
    - 데이터가 들어올 때마다 "야, 혹시 개수가 안 차더라도 윈도우 끝나는 시간(20초)이 되면 나를 깨워줘!"라고 Flink에게 알람을 맞춰두는 것입니다.

- **`on_processing_time`**
	- 위에서 맞춘 알람이 울리면 실행됩니다. "아, 개수는 못 채웠지만 시간은 됐네? 발사!" (`FIRE`)