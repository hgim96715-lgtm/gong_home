---
aliases:
  - Python Kafka Consumer
  - Kafka-python
  - NoBrokersAvailable
tags:
  - Kafka
  - Python
related:
  - "[[Kafka_Python_Producer]]"
  - "[[00_Kafka_HomePage]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[MinIO_Concept]]"
  - "[[Buffering_vs_Batching]]"
  - "[[Pandas_Read_Write]]"
---

# Kafka_Python_Consumer

## 개념 한 줄 요약

> **"우편함(Topic)을 주기적으로 확인해서 편지(Message)를 꺼내 읽는 구독자."** Producer 가 보낸 데이터를 가져와서 실제 로직(DB 저장, 분석, 알림 등)을 수행하는 주체.

---

---

# 왜 필요한가

```
Producer 가 아무리 데이터를 열심히 보내도
받아서 처리하는 놈이 없으면 Kafka 는 그냥 데이터 무덤

Kafka 는 데이터를 "밀어주는(Push)" 구조가 아님
Consumer 가 직접 "가져가는(Pull)" 구조
→ Consumer 를 띄워야 Topic 에 쌓인 데이터가 소비됨
```

---

---

# ① import

```python
from kafka import KafkaConsumer
from kafka.errors import NoBrokersAvailable  # 연결 실패 에러 처리용
import json
```

```
NoBrokersAvailable 을 import 하지 않으면
except NoBrokersAvailable: 에서 파이썬이 "그게 뭔데?" 하고 또 에러를 냄
```

---

---

# ② KafkaConsumer 생성 — 옵션 설정 ⭐️

```python
consumer = KafkaConsumer(
    "train-realtime",                # ① 구독할 토픽
    bootstrap_servers="kafka:9092",  # ② 접속 주소
    group_id="train-consumer-group", # ③ 컨슈머 그룹 ID
    auto_offset_reset="latest",      # ④ 처음 읽기 시작 위치
    enable_auto_commit=True,         # ⑤ 읽은 위치 자동 저장
    value_deserializer=lambda x: json.loads(x.decode("utf-8")),  # ⑥ 역직렬화
    consumer_timeout_ms=1000,        # ⑦ 데이터 없을 때 대기 시간
)
```

---

## ① 토픽 이름

```python
# 하나
KafkaConsumer("train-realtime", ...)

# 여러 개 동시 구독
KafkaConsumer("train-realtime", "train-schedule", ...)

# 정규표현식 (train- 으로 시작하는 것 전부)
import re
KafkaConsumer(pattern="train-.*", ...)
```

```
⚠️ Producer 코드에서 보낸 토픽 이름과 글자 하나, 띄어쓰기 하나까지 똑같아야 함
```

---

## ② bootstrap_servers — 접속 주소

```
맥북 터미널에서 실행할 때   →  localhost:29092  (도커 외부 포트)
도커 컨테이너 안에서 실행   →  kafka:9092       (도커 내부 네트워크)

주소가 틀리면 NoBrokersAvailable 에러 발생
Producer 코드에 적힌 주소와 동일하게 맞춰야 함

[[Docker_Compose_Setup]] 참고
```

---

## ③ group_id — 컨슈머 그룹 ⭐️⭐️⭐️

```
Consumer 들을 묶어주는 팀 이름
가장 중요한 옵션
```

```
같은 group_id 끼리  →  데이터를 N 빵 (분산 처리)
  Consumer 3개, 파티션 3개 → 각자 1개씩 맡아서 처리

다른 group_id 끼리  →  데이터를 복제해서 각자 받음 (Fan-out)
  A 팀도 받고, B 팀도 똑같이 받음

group_id 를 안 쓰면  →  Kafka 가 임의 ID 부여
  껐다 켜면 "새로운 사람" 으로 인식
  처음부터 다시 읽거나 데이터 놓칠 수 있음
```

```
Streamlit 실시간 대시보드에서는 group_id 생략 or None 권장

이유 ①: 브라우저 탭 3개 켜면 3개 화면 모두 동일한 데이터가 나와야 함
         (group_id 쓰면 데이터가 3등분 돼서 차트가 끊김)

이유 ②: 앱 껐다 켰을 때 과거 밀린 데이터 말고
         지금 들어오는 최신 데이터부터 보는 게 목적
```

---

## ④ auto_offset_reset — 어디서부터 읽을까

```
group_id 로 기록된 과거 내역(Offset) 이 없을 때의 행동 요령

"latest"    지금부터 들어오는 것만 줘 (실시간성 중요할 때)
"earliest"  맨 처음부터 싹 다 줘 (데이터 유실 절대 안 됨)
"none"      기록 없으면 그냥 에러내고 죽어
```

---

## ⑤ enable_auto_commit — 읽은 위치 자동 저장

```
데이터를 가져간 뒤 "나 여기까지 읽었음" 을 Kafka 에 기록(Commit)하는 방식

True  (기본값) : 5초마다 자동 체크
                 편하지만 처리 도중에 죽으면 데이터 유실 or 중복 가능

False          : consumer.commit() 을 직접 코드에 적어야 체크
                 귀찮지만 안전 (금융권 등 정합성 중요한 경우)
```

---

## ⑥ value_deserializer — 역직렬화 (번역기)

```
Kafka 는 데이터를 bytes 로 저장
이걸 파이썬이 쓸 수 있는 dict / str 로 변환해주는 번역기

설정 안 하면 b'{"trn_no": "00051"}' 처럼 바이트 덩어리 그대로 나옴
```

```python
# JSON → dict (가장 많이 씀)
value_deserializer=lambda x: json.loads(x.decode("utf-8"))

# 단순 문자열
value_deserializer=lambda x: x.decode("utf-8")
```

```
Producer value_serializer 와 대칭 관계
  Producer: dict → json.dumps → encode → bytes  (직렬화)
  Consumer: bytes → decode → json.loads → dict  (역직렬화)
```

---

## ⑦ consumer_timeout_ms — 대기 시간

```
기본값: 무한 대기 (데이터가 올 때까지 영원히 기다림)
설정값: 1000 = 1초

Streamlit 에서 필수:
  설정 안 하면 데이터가 끊겼을 때 화면 업데이트를 못 함
  1초 기다렸다가 없으면 루프 빠져나와서 UI 갱신하고 다시 시도
```

---

## 파라미터 한눈에 정리

|파라미터|필수|기본값|역할|
|---|:-:|---|---|
|`topic`|✅|—|구독할 채널|
|`bootstrap_servers`|✅|`localhost:9092`|브로커 주소|
|`group_id`|권장|랜덤 생성|소속 팀명|
|`auto_offset_reset`|선택|`latest`|처음 읽기 위치|
|`enable_auto_commit`|선택|`True`|읽은 위치 자동 저장|
|`value_deserializer`|선택|bytes 그대로|역직렬화 (번역기)|
|`consumer_timeout_ms`|선택|무한 대기|타임아웃|

---

---

# ③ 메시지 읽기 — 기본 루프

```python
for message in consumer:
    # message 의 구조
    print(message.topic)      # "train-realtime"
    print(message.partition)  # 0
    print(message.offset)     # 42
    print(message.key)        # None (key 없이 보냈을 때)
    print(message.value)      # {"trn_no": "00051", ...}  ← 역직렬화된 데이터
```

```
message.value 가 실제 데이터
나머지는 메타데이터 (어느 파티션의 몇 번째 메시지인지)
```

---

---

# ④ 배치 처리 — 버퍼에 모았다가 한 번에 ⭐️

```
건건이 DB/파일에 저장하면 I/O 가 너무 잦아서 시스템 부하

버퍼(메모리)에 일정량 모았다가 한 번에 처리 = Batch
속도 극대화 + 저장소 부하 감소
```

```python
buffer    = []    # 데이터를 임시로 모아둘 바구니
BATCH_SIZE = 100  # 100개가 모이면 한 번에 처리

for message in consumer:
    buffer.append(message.value)

    if len(buffer) >= BATCH_SIZE:
        # DB 저장 or Parquet 쓰기 등 처리 로직
        save_to_db(buffer)
        buffer.clear()   # 바구니 비우기

# 루프 끝나고 남아있는 것 처리 (100개 미만 잔여분)
if buffer:
    save_to_db(buffer)
    buffer.clear()
```

---

---

# ⑤ NoBrokersAvailable — 연결 재시도

```python
import time
from kafka.errors import NoBrokersAvailable

consumer = None
for attempt in range(1, 11):
    try:
        consumer = KafkaConsumer(
            "train-realtime",
            bootstrap_servers="kafka:9092",
            group_id="train-consumer-group",
            value_deserializer=lambda x: json.loads(x.decode("utf-8")),
        )
        print(f"✅ Kafka 연결 성공 ({attempt}회 시도)")
        break
    except NoBrokersAvailable:
        print(f"⏳ Kafka 연결 대기 중... ({attempt}/10)")
        time.sleep(3)

if consumer is None:
    raise RuntimeError("❌ Kafka 연결 실패. 컨테이너 상태를 확인하세요.")
```

---

---

# ⑥ 전체 패턴 — 연결 재시도 + 배치 처리

```python
from kafka import KafkaConsumer
from kafka.errors import NoBrokersAvailable
import json, time

# 1단계: 연결
consumer = None
for attempt in range(1, 11):
    try:
        consumer = KafkaConsumer(
            "train-realtime",
            bootstrap_servers="kafka:9092",
            group_id="train-consumer-group",
            auto_offset_reset="latest",
            enable_auto_commit=True,
            value_deserializer=lambda x: json.loads(x.decode("utf-8")),
            consumer_timeout_ms=1000,
        )
        print(f"✅ 연결 성공 ({attempt}회 시도)")
        break
    except NoBrokersAvailable:
        print(f"⏳ 대기 중... ({attempt}/10)")
        time.sleep(3)

if consumer is None:
    raise RuntimeError("Kafka 연결 실패")

# 2단계: 배치 처리
buffer     = []
BATCH_SIZE = 100

try:
    for message in consumer:
        data = message.value
        buffer.append(data)
        print(f"  수신: {data.get('trn_no')} | offset={message.offset}")

        if len(buffer) >= BATCH_SIZE:
            save_to_db(buffer)   # 실제 저장 로직
            buffer.clear()

except KeyboardInterrupt:
    print("\n🛑 종료")

finally:
    # 잔여 데이터 처리
    if buffer:
        save_to_db(buffer)
    consumer.close()
    print("Consumer 종료 완료")
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`NoBrokersAvailable`|Kafka 미실행 / 주소 틀림|`docker ps` 확인, `bootstrap_servers` 확인|
|데이터가 안 들어옴|`auto_offset_reset="latest"` 인데 이미 지나간 메시지|`"earliest"` 로 변경 or 새 메시지 발행|
|`b'{"trn_no"...}'` 바이트 그대로 나옴|`value_deserializer` 미설정|`lambda x: json.loads(x.decode("utf-8"))` 추가|
|같은 메시지가 여러 번 처리됨|`enable_auto_commit=False` 인데 commit 안 함|`consumer.commit()` 추가 or `True` 로 변경|
|Streamlit 화면 멈춤|`consumer_timeout_ms` 미설정|`consumer_timeout_ms=1000` 추가|
|데이터가 3개 탭에서 1/3 씩 나뉘어 나옴|Streamlit 에서 `group_id` 사용|`group_id=None` 으로 변경|