---
aliases:
  - Python Kafka Consumer
  - Kafka-python
  - NoBrokersAvailable
tags:
  - Kafka
  - Python
related:
  - "[[Kafka_Python_Producer_Basic]]"
  - "[[00_Kafka_HomePage]]"
  - "[[Docker_Host_vs_Internal_Network]]"
---
## Concept Summary

**"우편함(Topic)을 주기적으로 확인해서 편지(Message)를 꺼내 읽는 구독자(Subscriber)입니다."**
Producer가 보낸 데이터를 가져와서 실제 비즈니스 로직(DB 저장, 분석, 알림 발송 등)을 수행하는 주체입니다.

## 왜 필요한가? (Why)

**문제점:**

- Producer가 아무리 데이터를 열심히 보내도, 그걸 받아서 처리해 주는 놈이 없으면 Kafka는 그냥 **데이터 무덤**이 됩니다.
- Kafka는 데이터를 "밀어주는(Push)" 게 아니라, Consumer가 "가져가는(Pull)" 구조입니다.

**해결책:**

- **Consumer**를 띄워놔야 Topic에 쌓인 데이터를 하나씩 가져와서 소비(Consume)할 수 있습니다.
- **Consumer Group**을 설정하면 여러 Consumer가 일을 나눠서 처리하거나(병렬 처리), 똑같은 데이터를 각자 다르게 처리(Fan-out)할 수 있습니다.

---
## KafkaConsumer

```python
from kafka import KafkaConsumer
```

- **정체:** Kafka 토픽에서 데이터를 **"꺼내 읽는 도구(Class)"** 입니다.

## KafkaConsumer 내부 해부

```python
consumer = KafkaConsumer(
    'iot-average-topic',             # ① 구독할 채널 (Topic)
    bootstrap_servers='kafka:9092',  # ② 접속 주소 (Address)
    group_id='iot-consumer-group',   # ③ 소속 팀 (Team ID)
    auto_offset_reset='latest',      # ④ 읽기 시작 위치 (Start Point)
    enable_auto_commit=True,         # ⑤ 자동 체크 (Auto Save)
    value_deserializer=lambda x: x.decode('utf-8') # ⑥ 번역기 (Decoder)
    consumer_timeout_ms=1000 # ⑦ 대기 시간 (Timeout)
)
```

### ① `'iot-average-topic'` (Topic Name)

**역할:** 내가 데이터를 가져올 **주제(Topic)** 이름입니다.
**특징:**
- 문자열(`'topic'`)로 하나만 적어도 되고,
- 리스트(`['topic_A', 'topic_B']`)로 여러 개를 동시에 구독할 수도 있습니다.
- 정규표현식으로 `'iot-.*'` (iot로 시작하는 거 다 줘!)라고 쓸 수도 있습니다.

#### **⚠️ 주의 (Critical): Producer와 일치 필수!** 
Producer(데이터 생산자) 코드에서 보낼 때 설정한 토픽 이름과 **글자 하나, 띄어쓰기 하나까지 똑같아야** 합니다.


### ② `bootstrap_servers` (Connection)

- **의미:** "방송국(Kafka) 위치가 어디야?"
- **역할:** Kafka 클러스터에 접속하기 위한 **주소와 포트**입니다.
- **주의할 점:**
    - `'kafka:9092'`: Docker Compose 내부끼리 통신할 때 씁니다.
    - `'localhost:9092'`: 내 컴퓨터(Host)에서 파이썬을 켤 때 씁니다.
    - 이 주소가 틀리면 `NoBrokersAvailable` 에러가 뜹니다.

**핵심:** **Producer 소스코드에 적힌 주소와 똑같이 맞춰야 합니다.** 서로 다른 문(Port)을 바라보면 데이터를 못 찾습니다.

> server를 뭘 할지 모르겠다면 [[Docker_Host_vs_Internal_Network]] 참고 

### ③ `group_id` (Identity) ⭐️⭐️⭐️ (가장 중요)

- **의미:** "너 어느 팀 소속이야?"
- **역할:** Consumer들을 묶어주는 **팀 이름**입니다.
- **작동 원리:**
    - **같은 ID끼리:** 데이터를 **N빵(분산 처리)** 합니다. (Consumer가 3개고 파티션이 3개면, 각자 1개씩 맡아서 처리)
    - **다른 ID끼리:** 데이터를 **복제(Broadcast)** 해서 받습니다. (A팀도 받고, B팀도 똑같이 받음)
- **만약 이걸 안 쓰면?:** Kafka가 임의의 ID를 부여하는데, 껐다 켜면 "새로운 사람"으로 인식해서 **처음부터 다시 읽거나 데이터를 놓칠 수 있습니다.**

#### 💡 [꿀팁] Streamlit 실시간 대시보드 할 때는? (생략 권장)

- **안 쓰는 게 국룰!** (`group_id=None` 혹은 생략)
- **이유 1 (Broadcast):** 브라우저 탭을 3개 켰을 때, 3개 화면 모두 **똑같은 데이터가 온전히** 나와야 하기 때문입니다. 
	- (ID를 쓰면 데이터가 3등분 돼서 차트가 끊깁니다!)
- **이유 2 (Latest):** 앱을 껐다 켰을 때, 과거에 밀린 데이터 말고 **"지금 당장"** 들어오는 최신 데이터부터 보는 게 목적이기 때문입니다.

### ④ `auto_offset_reset` (Policy)

- **의미:** "처음 왔는데, 어디서부터 읽을까요?"
- **역할:** 이 `group_id`로 기록된 **과거 내역(Offset)이 없을 때**의 행동 요령입니다.
- **옵션:**
    - **`'latest'` (기본값):** "지나간 건 잊고, **지금부터** 들어오는 것만 줘." (실시간성 중요)
    - **`'earliest'`:** "맨 처음부터 **싹 다** 줘." (데이터 유실 절대 안 됨)
    - **`'none'`:** "기록 없으면 그냥 에러 내고 죽어."

### ⑤ `enable_auto_commit` (Consistency)

- **의미:** "읽었다고 체크(V)하는 걸 자동으로 할까?"
- **역할:** 데이터를 가져간 뒤에 "나 여기까지 읽었음"이라고 Kafka에 기록(Commit)하는 방식입니다.
- **옵션:**
    - **`True` (기본값):** 5초마다(기본설정) 알아서 체크합니다. **편하지만**, 처리 도중에 죽으면 데이터가 유실되거나 중복될 수 있습니다.
    - **`False`:** 내가 코드에서 `consumer.commit()`이라고 적어야만 체크합니다. **귀찮지만 안전**합니다. (금융권 등 정합성이 중요할 때 사용)

### ⑥ `value_deserializer` (Translator)

- **의미:** "010101로 된 거, 뭘로 번역해 줄까?"
- **역할:** Kafka는 데이터를 무조건 **바이트(Bytes)** 로 저장합니다. 이걸 파이썬이 쓸 수 있는 문자열이나 JSON으로 바꿔주는 **번역기**입니다.
- **자주 쓰는 패턴:**
    - **문자열로:** `lambda x: x.decode('utf-8')`
    - **JSON(딕셔너리)으로:** `lambda x: json.loads(x.decode('utf-8'))` (가장 많이 씀)
    - **이걸 안 쓰면?:** `b'hello'` 처럼 바이트 덩어리가 그대로 튀어나옵니다.

### ⑦ 대기 시간 (`consumer_timeout_ms`)  (Streamlit 필수)

- **의미:** **"데이터가 안 오면 얼마나 기다릴까?"**
- **기본값:** 무한 대기 (데이터가 올 때까지 영원히 기다림)
- **설정값:** `1000` (1000ms = 1초)

**왜 쓸까?**
- **UI 멈춤 방지:** 이걸 설정하지 않으면, 데이터가 끊겼을 때 Streamlit이 **"데이터 주세요..."** 하고 멍하니 기다리느라 화면 업데이트(새로고침)를 못 합니다.
- **루프 탈출:** 1초 동안 기다려보고 데이터가 없으면, `StopIteration` 에러를 내거나 루프를 빠져나와서 **"잠깐 쉬었다가(UI 갱신하고) 다시 올게"** 할 수 있게 해줍니다.


---

|**파라미터**|**필수 여부**|**안 쓰면 기본값**|**역할 비유**|
|---|---|---|---|
|**topic**|**필수**|(없음)|TV 채널 번호|
|**bootstrap_servers**|**필수**|`localhost:9092`|방송국 주소|
|**group_id**|**필수(권장)**|랜덤 생성|소속 팀명 (작업반)|
|**auto_offset_reset**|선택|`'latest'`|책갈피 없을 때 어디 펼칠지|
|**enable_auto_commit**|선택|`True`|읽은 페이지 자동 체크|
|**value_deserializer**|선택|(Bytes 그대로)|암호 해독기|

---
## `NoBrokersAvailable` (안전장치)

```python
from kafka.errors import NoBrokersAvailable
```

- **정체:** Kafka 서버(Broker)가 **"죽어있거나 연결이 안 될 때"** 발생하는 **특정 에러의 이름표**입니다.
- 이 이름을 import 하지 않으면, `except NoBrokersAvailable:` 이라고 썼을 때 Python이 _"NoBrokersAvailable이 뭔데? 영어 단어니?"_ 하고 또 에러를 냅니다.

---
## 실전 코드

```python
# 1. 도구 상자에서 꺼내오기 (Import)
from kafka import KafkaConsumer            # 수신기 꺼내오기
from kafka.errors import NoBrokersAvailable # '연결실패'라는 병명(Error Name) 알아오기

# 2. 실제 사용
try:
    # 수신기 조립 (KafkaConsumer 사용)
    consumer = KafkaConsumer('my_topic', bootstrap_servers=['localhost:9092'])

except NoBrokersAvailable: 
    # 연결 실패 병명 사용 (NoBrokersAvailable 사용)
    # "이 에러가 나면 프로그램 죽이지 말고 여기로 와!"
    print("지금 방송국(Broker)이 문을 닫았습니다. 잠시 대기합니다...")
```

