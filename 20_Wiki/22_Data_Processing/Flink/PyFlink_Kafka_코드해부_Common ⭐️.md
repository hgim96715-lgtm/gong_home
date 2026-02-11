---
aliases:
  - PyFlink Kafka Code Analysis
  - Kafka Source/Sink Explained
  - PyFlink Syntax
tags:
  - PyFlink
  - Kafka
related:
  - "[[PyFlink_Import_Analysis]]"
  - "[[PyFlink + Kafka 연동 완벽 가이드 ⭐️]]"
  - "[[PyFlink_코드 해부_common ⭐️]]"
  - "[[01_Apache Flink_Flow#**코드 작성 순서(Logic Flow)** 와 **데이터 흐름(Data Flow)**]]"
linked:
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/kafka_sink.py
---
# 🧐 PyFlink + Kafka: 코드 한 줄 한 줄 뜯어보기

> [!NOTE] 이 노트의 목적
> PyFlink에서 Kafka를 연결할 때 매번 복사/붙여넣기 하던 **Import 구문**과 **설정값**들이 도대체 **"왜"** 필요한지, 그리고 **"무슨 일"**을 하는지 명확하게 정리해서 헷갈릴때 마다 계속 보기 위함

---
##  Import 구문 완전 분석 (The Toolbox) 

이 친구들이 없으면 Kafka와 대화 자체가 불가능합니다.

###  기본 도구

```python
import os
from pyflink.common import SimpleStringSchema, WatermarkStrategy, SerializationSchema
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
```

- **`{scss}os`**: 파일 경로(`jar_path`)를 다룰 때 씀. (운영체제 기능)

- **`{scss}SimpleStringSchema`**: **(통역사 1)** 
	* **Why?** Kafka는 데이터를 `Byte`(010101)로 저장하고, 파이썬은 `String`("hello")으로 씁니다.
	- **Role:** 이 둘 사이를 변환해줍니다. (Byte ↔ String)

- **`{scss}WatermarkStrategy`**: **(시간 관리자)**
	- **Why?** 스트리밍 데이터는 순서가 뒤죽박죽일 수 있습니다.
	- **Role:** "데이터가 여기까지 도착했어!"라고 도장을 찍어주는 역할. (단순 조회만 할 땐 `no_watermarks()`를 써서 기능을 끔).

- **`{scss}Types`**: **(신호등/표지판)**
	- **Why?** 파이썬은 `var = 1` 했다가 `var = "hi"` 해도 되지만, Flink(자바)는 안 됩니다.
	- **Role:** `map` 같은 함수가 **"결과로 뭘 뱉는지(String인지 Int인지)"** Flink에게 미리 신고하는 역할. (안 하면 에러 남!)

#### 자주 쓰는 타입 매핑표 (Cheat Sheet)

파이썬 자료형을 Flink `Types`로 바꿀 때 참고하세요.

|**파이썬 코드 (return)**|**Flink 타입 (Types)**|**설명**|
|---|---|---|
|`return "hello"`|`Types.STRING()`|문자열|
|`return 100`|`Types.INT()`|정수 (약 ±21억)|
|`return 10000000000`|`Types.LONG()`|큰 정수 (ID, 시간 등)|
|`return 3.14`|`Types.DOUBLE()`|실수 (소수점)|
|`return (1, "A")`|`Types.TUPLE([Types.INT(), Types.STRING()])`|튜플 (순서 중요!)|
|`return [1, 2, 3]`|`Types.LIST(Types.INT())`|리스트 (모두 같은 타입)|

>**사용자 수, 개수:** `Types.INT()` (충분함)
>**타임스탬프(밀리초), DB의 ID값:** 무조건 **`Types.LONG()`** 을 쓰세요! (가장 많이 하는 실수입니다.)

- **`{scss}StreamExecutionEnvironment`**: **(작업장)**
	- **Role:** Flink 파이프라인(Source -> Process -> Sink)을 그리는 도화지.

### Kafka 전용 도구 (Connectors)

```python
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaSink, \
    KafkaRecordSerializationSchema, KafkaOffsetsInitializer
```

- **`{scss}KafkaSource`**: **(수도꼭지)** Kafka에서 데이터를 빨아들이는 입구 구현체.

- **`{scss}KafkaSink`**: **(배수구)** 처리된 데이터를 Kafka로 내보내는 출구 구현체.

- **`{scss}KafkaRecordSerializationSchema`**: **(포장 전문가)**
	- **Why?** 데이터를 Kafka로 보낼 때 "어느 토픽으로?", "Key는 뭐로?", "Value는 뭐로?" 보낼지 결정해야 함.
	- **Role:** Sink로 나가는 데이터의 포장 방식을 정의함.

- **`{scss}KafkaOffsetsInitializer`**: **(출발점 지정)**
	- **Role:** "처음부터 읽을래(`earliest`)?" 아니면 "지금부터 읽을래(`latest`)?"를 결정.

### Window & Time 도구 (The Slicers) 

무한한 데이터 스트림을 "시간"이나 "개수" 단위로 뚝뚝 끊어주는 도구들입니다.

```python
from pyflink.datastream.window import TumblingProcessingTimeWindows, SlidingProcessingTimeWindows, CountTrigger,Trigger, TriggerResult
from pyflink.common.time import Time
```

- **`{python}TumblingProcessingTimeWindows`**: **(일정한 간격의 칼)**
	- **Why?** 스트림은 끝이 없어서 합계(`sum`)를 못 구합니다. "10초마다 끊어서 계산해!"라고 명령해야 합니다.
	- **Role:** 시간을 기준으로 **겹치지 않게(Tumbling)**, **서버 시간(Processing Time)** 에 맞춰 데이터를 자르는 윈도우 생성기.
	- _참고:_ `EventTime`을 쓸 땐 `TumblingEventTimeWindows`로 바꿔야 함.

- **`{python}SlidingProcessingTimeWindows`**: **(미끄러지는 창문 - 스르륵)** 
	- **Why?** "지난 1분간의 평균을 **10초마다 갱신**해서 보여줘!" (데이터가 겹침)
	- **Role:** 윈도우를 옆으로 밀면서(Sliding) 생성함. **크기(Size)** 와 **갱신 주기(Slide)** 두 가지 설정이 필요함.

- **`{python}CountTrigger`**: **(개수 기준 방아쇠)**
	- **Why?** "1시간짜리 윈도우인데, 데이터가 100개 쌓이면 **미리** 계산해주면 안 돼?" (시간을 기다리기 지루할 때)
	- **Role:** 윈도우 안에 데이터가 **설정된 개수(N)** 만큼 찰 때마다 계산(`FIRE`)을 명령하는 트리거. (예: `CountTrigger.of(10)`)

- **`{python}Trigger`**: **(커스텀 트리거 설계도)**
	- **Why?** "데이터가 10개 모이거나(OR) 20초가 지나면 발사해!" 같은 **복합 조건**은 기본 트리거로 안 됩니다.
	- **Role:** 나만의 트리거를 만들 때 상속받아야 하는 **부모 클래스(Abstract Class)**. (`on_element` 등 필수 규칙 정의)

- **`{python}TriggerResult`**: **(판결 신호등)**
	- **Why?** 트리거가 조건을 확인한 후, Flink에게 "발사해!", "기다려!", "삭제해!"라고 **최종 명령**을 내려야 합니다.
	- **Role:** `FIRE`(발사), `CONTINUE`(대기), `PURGE`(삭제) 등 4가지 상태를 정의한 **열거형 상수(Enum)**.

>더보기 : 더 자세한 Trigger내용은 **[[PyFlink_Trigger_Watermark]]** 참조 

- **`{python}Time`**: **(시간 단위 변환기)**
	- **Why?** 컴퓨터에게 숫자 `10`만 주면 10초인지 10분인지 모릅니다.
	- **Role:** `Time.seconds(10)`, `Time.minutes(5)` 처럼 사람이 쓰는 시간을 Flink가 알아먹는 단위로 바꿔주는 도우미 클래스.

### Function & Logic 

윈도우 안에 모인 데이터를 실제로 요리(계산)하는 도구입니다.

```python
from pyflink.datastream.functions import AggregateFunction
```

- **`{python}AggregateFunction`**: **(고급 요리사 / 복합 계산기)**
	- **Why?** `reduce`(단순 더하기)만으로는 **평균(Average)** 같은 복잡한 계산을 못 합니다. (총점과 개수를 따로 기억해야 하니까요!)
	- **Role:** `create_accumulator`(그릇 준비), `add`(담기), `getResult`(계산) 단계를 직접 정의하여 **복잡한 로직을 처리**하는 클래스.


### State Management (The Memory) 

데이터가 흐르는 도중에도 **값을 기억(저장)** 해야 할 때 쓰는 도구입니다.

```python
from pyflink.datastream.state import ValueStateDescriptor
```

- **`{python}ValueStateDescriptor`**: **(기억 상자 명찰 / 사물함 신청서)**
	- **Why?** 파이썬 변수(`a = 1`)는 함수 실행이 끝나면 휘발되어 사라집니다. 하지만 "지금까지의 합계"나 "이전 데이터"를 **계속 기억**하려면 Flink에게 "저장 공간 좀 줘!"라고 해야 합니다.
	- **Role:** Flink에게 **"나 'count'라는 이름의 사물함(State) 하나 쓸게. 거기엔 '숫자(Int)'만 넣을 거야."** 라고 **스펙을 정의(`"이름", 타입`)해서 신청**하는 객체.
	- _사용 예시:_ `self.count_state_desc = ValueStateDescriptor("count", Types.INT())`

---
## Environment & JAR 설정 (The Bridge)

```python
env = StreamExecutionEnvironment.get_execution_environment()

# ⭐️ 여기가 제일 중요!
jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
env.add_jars(jar_path)
```

- `{scss}env.add_jars(jar_path)`
	- Q: 이게 뭔가요?
		- PyFlink(파이썬)가 Flink(자바)에게 "야, 나 Kafka랑 대화해야 되는데 통역사가 필요해"라고 말하며 **JAR 파일(라이브러리)** 을 건네주는 과정입니다.
	- Q: 왜 매번 경로를 지정하나요?
		- 파이썬 스크립트는 JAR가 어디 있는지 모르기 때문에, "여기 있어!"라고 명시적으로 찔러주는 것입니다.
	- **Q: 안 쓰면 어떻게 되나요?**
		- `ClassNotFoundException` 또는 `Java dependencies...` 에러가 나면서 실행 자체가 안 됩니다.

---
## Kafka Source 설정 (Input)

```python
source = KafkaSource.builder() \
    .set_bootstrap_servers("kafka:9092") \
    .set_topics("input-topic") \
    .set_group_id("my-flink-group") \
    .set_starting_offsets(KafkaOffsetsInitializer.earliest()) \
    .set_value_only_deserializer(SimpleStringSchema()) \
    .build()
```

- **`{scss}.set_bootstrap_servers("kafka:9092")`**: **(주소)**
	- Kafka 브로커가 어디 사는지 주소를 적습니다.
	- **주의:** Docker 내부 통신이므로 `localhost`가 아니라 **컨테이너 이름(`kafka`)** 을 써야 합니다.

- **`{scss}.set_topics("input-topic")`**: **(채널)**
	- 수많은 데이터 파이프 중 "어느 파이프의 물을 마실지" 결정합니다. (생성한 토픽 이름! )

- **`{scss}.set_group_id("my-flink-group")`**: **(신분증)**
	- Kafka에게 "나 `my-flink-group` 팀이야"라고 신원을 밝힘.
	- **Why?** Flink가 죽었다가 살아나도, Kafka가 "너 아까 100번까지 읽었어"라고 기억해 주기 위함(Offset 저장).

- **`{scss}.set_starting_offsets(...)`**: **(시작점)**
	- `earliest()`: 처음부터 싹 다 읽기 (과거 데이터 포함).
	- `latest()`: 지금 이 순간부터 들어오는 것만 읽기 (실시간).

- **`{scss}.set_value_only_deserializer(...)`**: **(해석기)**
	- Kafka 메시지는 `Key`, `Value`, `Header` 등이 있는데, "난 내용물(`Value`)만 필요해!" 라고 선언하고, 그걸 `String`으로 바꿔달라고 요청함.

---
## Source 연결 & Stream 생성 (The Connection)

만들어둔 소스(부품)를 작업장에 조립해서, **실제로 데이터가 흐르는 `stream` 객체**를 만드는 단계입니다. 
**이 한 줄이 없으면 `map` 함수를 아예 사용할 수 없습니다.**

```python
stream = env.from_source(
    source, 
    WatermarkStrategy.no_watermarks(), 
    "Kafka Source"
)
```

- **`{scss}env.from_source(...)`**: **(수도꼭지 설치 & 물 틀기)**
	- **Why?**: 위에서 만든 `source` 변수는 아직 "설정값 뭉치(객체)"일 뿐입니다. 이걸 실행 환경(`env`)에 등록해야 비로소 데이터가 흐르는 **Stream(강물)** 이 됩니다.
	- **Return (`stream`)**: 이제부터 이 `stream` 변수를 가지고 `map`, `filter`, `sink_to` 등을 수행합니다. (**Source 객체에는 `map` 함수가 없습니다!**)

- **`{scss}WatermarkStrategy.no_watermarks()`**: **(시간 관리 옵션)**
	- 실시간 데이터의 시간 흐름(Event Time)을 어떻게 다룰지 정하는 건데, 단순 데이터 처리나 로그 수집에서는 `no_watermarks()`(신경 안 씀)로 충분합니다.

- **`{scss}"Kafka Source"`**:
	-  나중에 Flink Web UI(대시보드) 그래프에서 보여질 이름표입니다.

----
## Transformation & TypeInfo (데이터 가공 & 타입 정의) 

이 부분이 빠지거나 틀리면 **"알 수 없는 리턴 타입"** 에러가 발생하며 Job이 시작조차 안 됩니다.
위에서 만든 'stream' 객체를 사용합니다!

### Why: 왜 굳이 타입을 적어야 하나요?

* **Python(자유로움):** "변수에 숫자 넣었다가 문자 넣어도 됨." (Dynamic Typing)
* **Flink/Java(엄격함):** "처음부터 크기(Byte)를 딱 정해놔야 직렬화(Serialization)해서 네트워크로 보낼 수 있음." (Static Typing)
* **결론:** 파이썬의 `lambda` 함수는 리턴 타입을 말해주지 않기 때문에, 우리가 **`output_type`** 파라미터로 **"야, 이거 결과물은 정수(Int)야!"** 라고 Flink에게 귀띔해줘야 합니다.

### The Golden Rule (공식)

> **`map`, `flatMap` 등 데이터를 변형하는 연산에는 무조건 `output_type`을 붙인다.**

###  Common Types Cheat Sheet (자주 쓰는 타입 모음)

가장 많이 쓰는 3가지 패턴, (Import 필수: `from pyflink.common.typeinfo import Types`)

#### Case 1. 문자열 (String).

가장 기본입니다. 텍스트를 다룰 때 사용합니다.

```python
# 위에서 만든 'stream' 객체를 사용합니다!
# stream.map(...) 은 "흐르는 강물에 변형을 가한다"는 뜻입니다.
# 예: 소문자로 변환
stream.map( # 👈 여기서 stream을 씁니다!
    lambda x: x.lower(), 
    output_type=Types.STRING()
)
```

#### Case 2. 숫자 (Integer, Long, Float)

계산 로직이 들어갈 때 사용합니다.

```python
# 예: 문자열 길이를 숫자로 반환 ("apple" -> 5)
stream.map(
    lambda x: len(x), 
    output_type=Types.INT() 
)
# 소수점이 필요하면 Types.FLOAT() 또는 Types.DOUBLE()
```

#### **Case 3. 튜플 (Tuple) ⭐️ (가장 많이 씀)**

파이썬의 `(값1, 값2)` 형태를 그대로 쓸 때 가장 간편합니다.

> 주의: Types.TUPLE([...]) 안에 리스트로 타입을 순서대로 넣어야 함!
>  **리스트(`[]`) 필수:** `Types.TUPLE` 안에 타입을 나열할 때는 반드시 대괄호(`[]`)로 감싸야 합니다. (안 감싸면 에러!)

```python
# 예: (단어, 1) 형태로 변환 -> ("apple", 1)
stream.map(
    lambda x: (x, 1),
    output_type=Types.TUPLE([Types.STRING(), Types.INT()]) 
)
# 주의: Types.TUPLE([...]) 안에 리스트로 타입을 순서대로 넣어야 함!
```

#### Case 4. 구조체 (Row / JSON) ⭐️ (데이터베이스 스타일)

실무에서는 단순 값보다 `{"name": "gong", "age": 20}` 같은 **JSON(Dictionary)** 형태를 많이 씁니다. 
이때는 **`Types.ROW`** 를 써야 합니다.
나중에 SQL이나 Table API랑 연동할 때 필수입니다.

> **리스트(`[]`) 필수:** `Types.ROW` 안에 타입을 나열할 때는 반드시 대괄호(`[]`)로 감싸야 합니다. (안 감싸면 에러!)

```python
from pyflink.common import Row

# 예: 이름과 나이를 분리해서 구조화된 데이터로 만듬
# 입력: "gong,20" -> 출력: Row(name="gong", age=20)
stream.map(
    lambda x: Row(x.split(',')[0], int(x.split(',')[1])),
    output_type=Types.ROW([       # 👈 반드시 리스트([])로 감싸야 함!
        Types.STRING(),           # 첫 번째 필드 타입
        Types.INT()               # 두 번째 필드 타입
    ])
)
```

### ⚠️ 주의사항

- **타입 불일치:** `Types.INT()`라고 선언해놓고 실제 코드에서 문자열("hello")을 리턴하면, 실행 중에 **ClassCastException**이 터집니다. (거짓말하면 안 됨!)

---
## Kafka Sink 설정 (Output)

```python
sink = KafkaSink.builder() \
    .set_bootstrap_servers("kafka:9092") \
    .set_record_serializer(
        KafkaRecordSerializationSchema.builder()
            .set_topic("output-topic")
            .set_value_serialization_schema(SimpleStringSchema())
            .build()
    ) \
    .build()
```

- **`{scss}.set_record_serializer(...)`**: **(포장지 설정)**
	- 데이터를 내보낼 때 어떻게 포장할지 아주 상세하게 정하는 부분.
	- `{scss}.set_topic("output-topic")` : 어느 토픽으로 쏠지 지정. (Source에서 읽은 것과 다른 토픽이어야 무한 루프 안 돔!!!!)(미리 만들어 두기)
	- **`{scss}.set_value_serialization_schema(...)`**: 파이썬의 `String` 데이터를 다시 Kafka가 좋아하는 `Byte`로 변환(`Serialization`)해서 포장함.


[!WARNING] **토픽 자동 생성 vs 수동 생성** 
>`docker-compose.yml`의 `KAFKA_AUTO_CREATE_TOPICS_ENABLE=true` 덕분에 토픽을 안 만들어도 에러는 안 나지만, **파티션이 1개**로 생성됩니다.
> **문제점:** 나중에 데이터가 많아져서 병렬 처리를 하려고 해도 파티션이 1개면 병목 현상이 생김. 
>  **권장:** `kafka-topics.sh` 명령어로 **파티션 3개 이상**으로 수동 생성하는 습관을 들이자.


---
## 요약 (Cheat Sheet)
| **코드**                  | **역할**            | **비유**                  |
| ----------------------- | ----------------- | ----------------------- |
| `Types.STRING()`        | 리턴 타입 명시          | "이 상자엔 문자열이 들어있습니다" 라벨링 |
| `SimpleStringSchema`    | Byte ↔ String 변환기 | 번역기 (0101 ↔ "안녕")       |
| `add_jars(...)`         | Kafka 커넥터 JAR 로딩  | 통역사 채용                  |
| `set_bootstrap_servers` | Kafka 주소 지정       | 내비게이션 목적지               |
| `set_group_id`          | 컨슈머 그룹 ID         | 회원 번호 (적립금/위치 저장용)      |
| `earliest()`            | 처음부터 읽기           | 정주행 (1화부터)              |
| `latest()`              | 지금부터 읽기           | 본방 사수 (지금 방송부터)         |
