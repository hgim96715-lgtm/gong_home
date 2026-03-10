---
aliases:
  - 카프카파이썬연동
  - KafkaProducer
  - NoBrokersAvailable
tags:
  - Kafka
  - Python
related:
  - "[[IOT_sensor_Flink]]"
  - "[[00_Kafka_HomePage]]"
  - "[[Apache_Kafka_Concept]]"
  - "[[Python_JSON]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[Kafka_CLI_Cheatsheet]]"
  - "[[Kafka_Python_Consumer]]"
---


# Kafka_Python_Producer

## 개념 한 줄 요약

> **파이썬 데이터를 Kafka 브로커(서버)로 쏘아 보내는 객체.** `NoBrokersAvailable` = "카프카 문 닫혀있는데?" — 연결 실패를 알리는 에러.

---

---

# 왜 필요한가

```
Flink / Spark 가 데이터를 처리하려면
일단 누군가가 Kafka 에 데이터를 넣어줘야 함
파이썬이 그 역할을 가장 쉽게 할 수 있음

Docker 환경에서는
Kafka 가 켜지는 속도보다 파이썬이 실행되는 속도가 더 빠를 때가 많음
→ NoBrokersAvailable 에러 발생
→ 그냥 죽지 않고 "기다렸다가 다시 연결" 로직이 필요
```

---

---

# ① 설치 및 import

```bash
pip install kafka-python
```

```python
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable  # 연결 실패 에러 처리용
import json
import time
```

---

---

# ② KafkaProducer 생성 — 옵션 설정 ⭐️

```python
producer = KafkaProducer(
    bootstrap_servers="kafka:9092",
    value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
    acks="all",    # 전송 보장 수준
    retries=3,     # 전송 실패 시 재시도 횟수
)
```

---

## bootstrap_servers — 어디로 보낼까?

```
카프카 브로커의 주소 (host:port)
하나만 적어줘도 알아서 클러스터 전체 정보를 찾아냄 (Bootstrap)
```

```
맥북 터미널에서 실행할 때  →  localhost:29092  (도커 외부 연결용 포트)
도커 컨테이너 안에서 실행할 때  →  kafka:9092  (도커 내부 네트워크)

헷갈리면 [[Docker_Compose_Setup]] 참고
```

---

## value_serializer — 어떻게 포장할까? (직렬화)

```
Kafka 는 오직 bytes 만 알아들음
Python dict {"a": 1} 을 그대로 던지면 에러

변환 순서:
  Python dict
      ↓ json.dumps(v, ensure_ascii=False)
  JSON 문자열  '{"a": 1}'
      ↓ .encode("utf-8")
  bytes  b'{"a": 1}'
      ↓ Kafka 로 전송
```

```python
# JSON 데이터 (가장 많이 씀)
value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8")
#                                        ↑ 한글이 \uXXXX 로 깨지지 않게

# 단순 문자열 데이터
value_serializer=lambda v: v.encode("utf-8")
# json.dumps 불필요. 핵심은 무조건 bytes 로 만들어야 한다는 것
```

---

## acks — 전송 보장 수준

|값|동작|특징|
|---|---|---|
|`0`|서버 응답 안 기다림|가장 빠름. 유실 위험|
|`1`|리더 파티션에 저장되면 성공|기본값|
|`"all"`|모든 replica 에 저장되면 성공|가장 안전. 데이터 파이프라인 권장|

---

---

# ③ 데이터 전송 — send() / flush()

```python
# 토픽으로 메시지 발행
producer.send("train-realtime", value={"trn_no": "00051", "stn_nm": "서울"})

# 버퍼에 남아있는 메시지 전부 전송 완료될 때까지 대기
producer.flush()
```

```
send() 는 비동기 (Async)
  "보내!" 하고 바로 다음 줄로 넘어감
  내부 버퍼에 쌓아두고 백그라운드에서 전송

flush() 는 동기 (Sync)
  버퍼에 남아있는 메시지를 전부 보낼 때까지 기다림
  루프 한 바퀴 끝난 후 or 프로그램 종료 전에 반드시 호출
  안 하면 프로그램 종료 시 버퍼 메시지 유실 가능
```

## future.get() — 전송 결과 확인

```python
future = producer.send("train-realtime", value=data)

# 전송 완료될 때까지 최대 10초 대기 후 결과 반환
record_metadata = future.get(timeout=10)

print(record_metadata.topic)      # train-realtime
print(record_metadata.partition)  # 0
print(record_metadata.offset)     # 42  ← 몇 번째 메시지인지
```

```
디버깅할 때 offset 확인하면 메시지가 정상적으로 들어갔는지 알 수 있음
고성능이 필요하면 future.get() 제거 (완전 비동기)
단, 에러 발생 여부를 즉시 알 수 없음
```

---

---

# ④ 연결 재시도 — NoBrokersAvailable 처리 ⭐️

```
Docker 환경에서 자주 발생:
  docker-compose up 하면 Kafka 가 완전히 켜지기 전에
  파이썬 컨테이너가 먼저 실행되어 연결 시도
  → NoBrokersAvailable 에러
```

```python
from kafka.errors import NoBrokersAvailable
import time

# 최대 10번, 3초 간격으로 재시도
producer = None
for attempt in range(1, 11):
    try:
        producer = KafkaProducer(
            bootstrap_servers="kafka:9092",
            value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
        )
        print(f"✅ Kafka 연결 성공 (시도 {attempt}회)")
        break
    except NoBrokersAvailable:
        print(f"⏳ Kafka 연결 대기 중... ({attempt}/10)")
        time.sleep(3)

if producer is None:
    raise RuntimeError("❌ Kafka 연결 실패. 컨테이너 상태를 확인하세요.")
```

---

---

# ⑤ 전체 패턴 — 연결 재시도 + 폴링 루프

```python
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable
import json, time

# 1단계: 연결 (최대 10번 재시도)
producer = None
for attempt in range(1, 11):
    try:
        producer = KafkaProducer(
            bootstrap_servers="kafka:9092",
            value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
            acks="all",
            retries=3,
        )
        print(f"✅ 연결 성공 ({attempt}회 시도)")
        break
    except NoBrokersAvailable:
        print(f"⏳ 대기 중... ({attempt}/10)")
        time.sleep(3)

if producer is None:
    raise RuntimeError("Kafka 연결 실패")

# 2단계: 폴링 루프 (무한 반복)
try:
    while True:
        data = {"message": "hello", "ts": time.time()}
        producer.send("my-topic", value=data)
        producer.flush()   # 루프마다 flush
        print(f"발행: {data}")
        time.sleep(1)      # 1초 간격

except KeyboardInterrupt:
    print("\n🛑 종료")

finally:
    producer.flush()   # 마지막 잔여 메시지 전송
    producer.close()   # 연결 정리
```

---

---

# ⑥ for - else 문법 (파이썬 특이점)

```python
for attempt in range(10):
    try:
        producer = KafkaProducer(...)
        break        # 성공하면 탈출 → else 블록 건너뜀
    except NoBrokersAvailable:
        time.sleep(3)
else:
    # break 없이 for 문이 끝까지 다 돌았을 때 실행됨
    # = 10번 시도 전부 실패했을 때
    raise RuntimeError("연결 완전 실패")
```

```
파이썬 for - else 규칙:
  break 로 빠져나오면  → else 실행 안 됨
  끝까지 다 돌면       → else 실행됨

"완전 실패" 처리에 깔끔하게 쓸 수 있음
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`NoBrokersAvailable` 계속 뜸|Kafka 미실행 / 주소 틀림|`docker ps` 확인, `bootstrap_servers` 주소 확인|
|`localhost:9092` 연결 안 됨|도커 안에서 실행 중|컨테이너 안에서는 `kafka:9092` 사용|
|전송했는데 Kafka 에 데이터 없음|`flush()` 미호출|루프 끝에 `producer.flush()` 추가|
|한글 깨짐|`ensure_ascii=True` (기본값)|`json.dumps(v, ensure_ascii=False)`|
|`TypeError: expected bytes`|`value_serializer` 미설정|`lambda v: json.dumps(v).encode("utf-8")` 추가|