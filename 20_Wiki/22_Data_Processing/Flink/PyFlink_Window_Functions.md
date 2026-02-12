---
aliases:
  - window Functions
  - AggregateFunction
  - ReduceFunction
  - ProcessWindowFunction
tags:
  - PyFlink
related:
  - "[[PyFlink_Windows]]"
  - "[[PyFlink_Operators_Basic]]"
  - "[[00_Apache Flink_HomePage]]"
---
#  PyFlink Window Functions: 윈도우 데이터 요리하기

> [!QUOTE] 핵심 요약
> **"윈도우(양동이)에 모인 데이터를 합치거나, 평균을 내거나, 분석해서 최종 결과를 만드는 단계."**
> * **Window:** 데이터를 모으는 역할 (Time, Count)
> * **Window Function:** 모인 데이터를 계산하는 역할 (Sum, Avg, Max)

---
##  Why? 왜 필요한가요? 

`window()`만 쓰면 데이터를 모아두기만 할 뿐, 아무런 일도 일어나지 않습니다.
Flink에게 **"10초치 데이터를 모았어? 그럼 이제 이걸로 평균을 구해(Average)!"** 라고 구체적인 계산법을 알려줘야 합니다.

---
## 핵심 함수 3대장 완전 해부 (The Big Three) 

Flink가 윈도우 데이터를 처리하는 3가지 방법입니다. 
난이도와 유연성에 따라 골라 써야 합니다.

### ① ReduceFunction (초급: "계속 더하기") 🥉

가장 단순하고 빠릅니다. 데이터가 들어올 때마다 즉시 합칩니다.

- **역할:** 입력값 2개를 받아서, 1개로 합쳐서 반환합니다. (1+1=2, 2+3=5...)
- **용도:** 합계(`sum`), 최댓값(`max`), 최솟값(`min`) 처럼 단순 누적 계산.

- **비유 (저금통):**
> "동전을 넣을 때마다 머릿속으로 합계를 갱신합니다. 100원 넣고, 500원 넣으면 '600원'만 기억합니다. 각각의 동전이 언제 들어왔는지는 까먹습니다."

- **구조 (Lambda):**

```python
# a: 누적된 값, b: 새로 들어온 값
lambda a, b: a + b
```

- **특징:** 입력 타입과 출력 타입이 무조건 같아야 합니다. (Int + Int = Int)

---
### ② AggregateFunction (중급: "중간 요리") 🥈

가장 유연하고 강력합니다. 입력과 출력 타입이 다를 때(예: 평균 구하기) 사용합니다

**반드시 아래 4단계를 구현해야 합니다.**

1. **`create_accumulator` (그릇 준비) 
    - **역할:** 계산을 시작하기 전, **빈 그릇(변수)** 을 만듭니다.
    - **비유:** "자, 평균 구할 거니까 (총점 0점, 개수 0개) 적힌 빈 종이 가져와."
    - **코드:** `return (0, 0)`

2. **`add` (재료 담기)**
    - **역할:** 데이터가 **하나 들어올 때마다** 그릇의 내용을 갱신합니다.
    - **비유:** "Alice 100점 들어왔네? (value) / 기존 총점 0점이었지? (accumulator) -> 합쳐서 100점으로 바꿔!"   
    - **문법(Syntax):** `def add(self, value, accumulator):` 👈 **(약속된 순서)**
	    - **`value`**: 방금 들어온 따끈따끈한 **새 데이터** (Flink가 알아서 첫 번째 자리에 넣어줌)
	    - **`accumulator`**: 지금까지 계산된 **중간 결과 바구니** (Flink가 알아서 두 번째 자리에 넣어줌)
	- **코드:**
```python
# (기존 총점 + 새 점수, 기존 개수 + 1)
return (accumulator[0] + value[1], accumulator[1] + 1)
```

3. **`get_result` (요리 완성)** 
    - **역할:** 윈도우가 끝날 때, 그릇에 담긴 내용으로 **최종 결과**를 계산해서 내보냅니다.
    - **비유:** "10초 지났네? 이제 (총점 / 개수) 계산해서 평균 점수만 내보내."
    - **코드:** `return 총점 / 개수`

4. **`merge` (냄비 합치기)**
    - **역할:** (세션 윈도우 등에서) 서로 다른 윈도우(냄비)가 합쳐질 때, 두 그릇을 어떻게 합칠지 정합니다.
    - **비유:** "어? 1번 냄비랑 2번 냄비랑 합쳐야 하네? 총점끼리 더하고 개수끼리 더해!"
    - **코드:** `return (총점A+총점B, 개수A+개수B)`

---
## `ProcessWindowFunction` ?

- `Reduce`가 들어오는 족족 계산하는 "성격 급한 녀석"이라면,
- `Process`는 **"윈도우 닫힐 때까지 기다렸다가, 데이터를 몽땅 받아서 한 번에 처리하는 신중한 녀석"** 입니다.

### 함수의 기본 골격 (Anatomy)

이 함수는 반드시 **`process`** 라는 메서드를 구현해야 하며, Flink가 4가지 재료를 넣어줍니다.

```python
class MyProcessFunction(ProcessWindowFunction):
    
    # key: 이 윈도우의 주인 (예: "Alice")
    # context: 윈도우의 시간 정보 (시작 시간, 끝 시간 등)
    # elements: 윈도우에 모인 "모든 데이터" (여기가 핵심!)
    def process(self, key, context, elements):
        # ... 로직 구현 ...
        yield 결과
```

#### `elements`: "리스트가 아니라 `Iterable` (반복자) 객체입니다."

**특징:**
- 자판기처럼 **"하나씩 꺼낼 수만"** 있습니다. (`next()`)
- 전체 길이가 몇 개인지(`len()`), 뒤에 뭐가 있는지 미리 알 수 없습니다.
- **메모리 절약**을 위해 Flink가 데이터를 아직 다 꺼내놓지 않은 상태입니다.
**변환 필수:**
- `data_list = list(elements)` 또는 `[e for e in elements]` 를 통해 리스트로 변환!
- **경고:** 윈도우 안에 데이터가 100만 개라면? 리스트로 바꾸는 순간 100만 개가 RAM에 올라가서 **메모리 폭발(OOM)** 이 일어날 수 있습니다. (그래서 `KeepLastN` 로직은 윈도우 사이즈가 작을 때만 써야 합니다.)


#### `yield`: "return 쓰면 해고당합니다"

- 일반적인 함수는 `return`을 쓰지만, Flink 함수들은 **제너레이터(Generator)** 패턴인 `yield`를 씁니다.
- Flink는 유연성을 위해 무조건 `yield`를 쓰도록 설계되었습니다.

#### `key` & `context`: "주소와 시간"

- **`key`:** 지금 처리하는 윈도우의 주인님입니다. (예: `key_by(x[0])` 했다면 "Alice").
- **`context`:** 윈도우의 **메타데이터**입니다.
	- **`context.window().start`**: 윈도우 시작 시간 (Unix Timestamp, **밀리세컨드 단위**)
	-  **`context.window().end`**: 윈도우 종료 시간 (Unix Timestamp, **밀리세컨드 단위**)
	- `context.current_watermark`: 현재 워터마크 시간


---
## `aggregate` 함수의 문법 (Type Hinting)

데이터의 **"생김새(Type)"** 를 미리 알려주는 필수 절차입니다.
무조건 우리가 짠 코드(`return` 값)와 일치해야 합니다.

```python
.aggregate(
    func,              # 1. 요리사 (Function)
    accumulator_type,  # 2. 중간 그릇의 생김새 (State Type)
    output_type        # 3. 완성된 요리의 생김새 (Result Type)
)
```

### ① `accumulator_type` (중간 그릇 신고)

- **연결 고리:** `create_accumulator()`와 `add()`가 리턴하는 값의 형태.
- **예시:**
	- 코드: `return (0, 0)` -> 정수 2개짜리 튜플
	- 타입: `Types.TUPLE([Types.INT(), Types.INT()])`

### ② `output_type` (최종 결과 신고)

- **연결 고리:** `get_result()`가 리턴하는 값의 형태.
- **예시:**
	- 코드: `return 150.0` -> 실수(소수점) 1개
	- 타입: `Types.DOUBLE()`

---
### 자주 쓰는 타입 매핑표 (Cheat Sheet)

파이썬 자료형을 Flink `Types`로 바꿀 때 참고하세요.

|**파이썬 코드 (return)**|**Flink 타입 (Types)**|**설명**|
|---|---|---|
|`return "hello"`|`Types.STRING()`|문자열|
|`return 100`|`Types.INT()`|정수 (약 ±21억)|
|`return 10000000000`|`Types.LONG()`|큰 정수 (ID, 시간 등)|
|`return 3.14`|`Types.DOUBLE()`|실수 (소수점)|
|`return (1, "A")`|`Types.TUPLE([Types.INT(), Types.STRING()])`|튜플 (순서 중요!)|
|`return [1, 2, 3]`|`Types.LIST(Types.INT())`|리스트 (모두 같은 타입)|


---
## Practical Code: Kafka + AggregateFunction (Average) 

Java의 `AverageAggregate` 클래스를 PyFlink로 변환했습니다.
입력으로 `("Alice", 100)` 점수가 들어오면, 10초간 모아서 **평균 점수**를 계산합니다.

```python
"""
[PyFlink] Window Average Example (AggregateFunction)
목표: Kafka 데이터를 30초간 모아서 '평균 값'을 계산한다.
입력 예시: "Alice,100" (CSV 스타일 문자열)
출력 예시: "window Result -> 150.0"
"""

import os
import ast
from pyflink.common import SimpleStringSchema, WatermarkStrategy
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaSink, KafkaRecordSerializationSchema, KafkaOffsetsInitializer
from pyflink.datastream.window import TumblingProcessingTimeWindows
from pyflink.common.time import Time
# ⭐️ AggregateFunction 위치 (최신 버전)
from pyflink.datastream.functions import AggregateFunction

# ---------------------------------------------------------
# 1. 평균 계산 로직 클래스 (The Chef) 🍳
# ---------------------------------------------------------
class AverageAggregate(AggregateFunction):
    
    # ① 초기화: (총점, 개수)를 저장할 빈 바구니 생성
    # Flink가 계산을 시작할 때 딱 한 번 호출합니다.
    def create_accumulator(self):
        return (0, 0) # (sum, count)-> 정수, 정수
    
    # ② 더하기: 데이터가 들어올 때마다 호출됨 🥕
    # value: ("Alice", 100) -> 튜플의 1번째 요소가 점수
    # accumulator: (현재총점, 현재개수) -> 지금까지 모은 상태
    def add(self, value, accumulator):
        # (기존총점 + 새점수, 기존개수 + 1)
        return (accumulator[0] + value[1], accumulator[1] + 1)
    
    # ③ 결과 도출: 윈도우가 닫힐 때 최종 계산 🍲
    # 총점 / 개수 = 평균
    # 150.5 리턴 -> 실수(Double)
    def get_result(self, accumulator):
        # 0으로 나누기 방지 (데이터가 없으면 0.0 리턴)
        return accumulator[0] / accumulator[1] if accumulator[1] > 0 else 0.0
    
    # ④ 병합: 여러 윈도우(파티션) 결과를 합칠 때 사용 🥘
    def merge(self, a, b):
        return (a[0] + b[0], a[1] + b[1])
    
# ---------------------------------------------------------
# 2. 메인 파이프라인 (The Factory) 🏭
# ---------------------------------------------------------
def run_window_average():
    
    # 1. 환경 설정
    env = StreamExecutionEnvironment.get_execution_environment()
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)
    
    # 2. Source 설정 (Kafka Input)
    source = (
        KafkaSource
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_topics("input-topic")
        .set_group_id("avg-group")
        .set_starting_offsets(KafkaOffsetsInitializer.latest())
        .set_value_only_deserializer(SimpleStringSchema())
        .build()
    )
    
    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")
    
    # 3. Map: 데이터 파싱 (CSV 스타일)
    # 입력: "Alice,100" (String)
    # 출력: ("Alice", 100) (Tuple)
    # 쉼표(,)를 기준으로 쪼개서 이름과 점수로 분리합니다.
    mapped_stream = stream.map(
        lambda x: (x.split(",")[0], int(x.split(",")[1])),
        output_type=Types.TUPLE([Types.STRING(), Types.INT()])
    )

    # 4. KeyBy + Window + Aggregate (핵심 로직)
    avg_stream = (
        mapped_stream
        .key_by(lambda x: x[0]) # 이름(Alice) 기준으로 모으기
        .window(TumblingProcessingTimeWindows.of(Time.seconds(30))) # 30초 동안 가두기
        .aggregate(
            AverageAggregate(), # 👈 우리가 만든 요리법 적용!
            # ① create_accumulator가 (Int, Int)니까 -> TUPLE([INT, INT])
            accumulator_type=Types.TUPLE([Types.INT(), Types.INT()]), 
            # ② get_result가 float니까 -> DOUBLE
            output_type=Types.DOUBLE() # 최종 결과 타입 (평균 실수값)
        )
    )
    
    # 5. Sink 설정 (Kafka Output)
    sink = (
        KafkaSink
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_record_serializer(
            KafkaRecordSerializationSchema.builder()
            .set_topic("output-topic")
            .set_value_serialization_schema(SimpleStringSchema())
            .build()
        )
    ).build()
    
    # 6. 결과 전송
    # 출력 포맷: "window Result -> 150.0"
    avg_stream.map(
        lambda x: f"window Result -> {x}",
        output_type=Types.STRING()
    ).sink_to(sink)
    
    env.execute("Kafka Window Average Example")
    

if __name__=="__main__":
    run_window_average()
```

---
