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
  - "[[Kafka_Error_Handling_Retry]]"
  - "[[Python_DateTime]]"
---


# Kafka_Python_Producer

## 개념 한 줄 요약

> **파이썬 데이터를 Kafka 브로커(서버)로 쏘아 보내는 객체.** 
> `NoBrokersAvailable` = "카프카 문 닫혀있는데?" — 연결 실패를 알리는 에러.

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
>[[Docker_Compose_Setup]] 
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

## send() 파라미터 전체

```python
producer.send(
    topic,            # ① 필수: 보낼 토픽 이름
    value=None,       # ② 실제 데이터 (value_serializer 거쳐서 bytes 변환)
    key=None,         # ③ 메시지 키 (파티션 라우팅 / 순서 보장)
    headers=None,     # ④ 메타데이터 헤더 (추적 ID 등)
    partition=None,   # ⑤ 파티션 직접 지정 (보통 안 씀)
    timestamp_ms=None # ⑥ 타임스탬프 (None 이면 Kafka 가 자동 부여)
)
```

---

## ① value — 실제 데이터

```python
# 가장 기본적인 사용법
producer.send("train-realtime", value={"trn_no": "00051", "stn_nm": "서울"})

# value_serializer 설정했으면 dict 그대로 넣어도 됨
# 내부에서 자동으로 bytes 변환
```

---

## ② key — 파티션 라우팅 / 순서 보장 ⭐️

```python
# key 도 bytes 로 넣어야 함
producer.send(
    "train-realtime",
    key=b"00051",                                      # 단순 bytes
    key="00051".encode("utf-8"),                       # 문자열 → bytes
    value={"trn_no": "00051", "stn_nm": "서울"}
)
```

```
key 의 역할:
  같은 key → 항상 같은 파티션으로 전달 (순서 보장)
  key=None → 라운드로빈으로 파티션에 분배

실전 활용:
  key="열차번호" → 같은 열차의 메시지가 항상 같은 파티션에 쌓임
  → Consumer 에서 열차별 순서대로 처리 가능

파티션이 1개면 key 의미 없음
  멀티 파티션 환경에서 의미 있음
```

---

## ③ headers — 메타데이터 전달

```python
# 추적 ID, 출처 정보 등 메타데이터를 함께 전송
producer.send(
    "train-realtime",
    value={"trn_no": "00051"},
    headers=[
        ("source", b"seoul-station-api"),
        ("version", b"1.0"),
    ]
)
# headers 는 (이름, bytes) 튜플의 리스트
```

```
Consumer 에서 꺼내기:
  for key, val in message.headers:
      print(key, val.decode("utf-8"))

실무에서는 데이터 출처 추적, 버전 관리에 활용
간단한 파이프라인에서는 안 써도 됨
```

---

## ④ partition — 파티션 직접 지정

```python
# 특정 파티션에 강제로 넣기 (거의 안 씀)
producer.send("train-realtime", value=data, partition=0)
```

```
보통은 key 로 파티션을 간접 지정함
partition 을 직접 쓰면 특정 파티션에 부하 몰릴 수 있음
```

---
## ⑤ value 안에 수집 시각 넣기 — isoformat() ⭐️

```
Kafka 메시지에 "이 데이터를 언제 수집했는지" 를 함께 담는 패턴
timestamp_ms 는 Kafka 내부 메타데이터 타임스탬프 (Consumer 가 꺼내기 불편)
collected_at 을 value 안에 직접 넣는 방식이 더 실용적
```


```python
from datetime import datetime

data = {
    "trn_no"      : "00051",
    "stn_nm"      : "서울",
    "collected_at": datetime.now().isoformat(),   # ← 여기
}

producer.send("train-realtime", value=data)
```


```python
# isoformat() 결과
datetime.now().isoformat()
# "2026-02-24T21:06:32.123456"
#            ↑
#            T 가 날짜와 시간 구분자

# T 를 공백으로 바꾸고 싶을 때
datetime.now().isoformat(sep=" ")
# "2026-02-24 21:06:32.123456"
```

```
isoformat() 을 쓰는 이유:
  사전순 정렬 = 시간순 정렬 (BigQuery, S3, Spark 정렬 보장)
  마이크로초까지 포함 → 중복 없이 고유한 시각 표현
  strftime 포맷 지정 없이 한 줄로 끝남

UTC 기준 수집 시각 (글로벌 서비스, 서버 배포 시 권장)
  from datetime import datetime, timezone
  datetime.now(timezone.utc).isoformat()
  # "2026-02-24T12:06:32.123456+00:00"  ← 끝에 +00:00 붙음
```

>[[Python_DateTime#방법 1 — isoformat() 국제 표준 ISO 8601|isoformat(날짜->문자열)]] 참고

---

---

## send() / flush() / close() 동작 원리

```scss
send()   비동기 (Async)
  "보내!" 하고 바로 다음 줄로 넘어감
  내부 버퍼에 쌓아두고 백그라운드에서 전송

flush()  동기 (Sync)
  버퍼에 남아있는 메시지를 전부 보낼 때까지 기다림
  루프 한 바퀴 끝난 후 반드시 호출
  안 하면 프로그램 종료 시 버퍼 메시지 유실 가능

close()  연결 종료
  flush() 를 내부적으로 한 번 더 호출한 뒤 Kafka 연결을 끊음
  네트워크 소켓 / 스레드 등 자원을 반납
  프로그램 종료 시 반드시 호출
```

```text
close() 안 하면?
  Kafka 브로커 입장에서 연결이 살아있다고 착각
  → 연결 수 초과, 리소스 낭비
  → 다음 실행 시 연결이 꼬이는 현상 발생 가능

flush() 만 하고 close() 안 하면?
  메시지 유실은 없지만 소켓 연결이 남아 있음
  → 장기 운영 시 리소스 누수
```

```python
# 올바른 종료 패턴 — finally 블록에서 처리
try:
    while True:
        producer.send("train-realtime", value=data)
        producer.flush()
        time.sleep(60)
except KeyboardInterrupt:
    print("🛑 종료")
finally:
    producer.flush()    # 버퍼에 남은 메시지 마지막으로 전송
    producer.close()    # 연결 정리 (내부적으로 flush 한 번 더 호출)
```

```text
finally 블록을 쓰는 이유:
  KeyboardInterrupt (Ctrl+C) 뿐만 아니라
  예외가 발생해도 반드시 close() 가 실행되도록 보장
  close() 없이 강제 종료하면 자원 정리가 안 됨
```

```python
# 루프마다 flush
for item in items:
    producer.send("train-realtime", value=item)
producer.flush()   # 루프 끝나고 한 번에 flush (루프마다 호출하면 비효율)
```

---

## future.get() — 전송 결과 확인

```python
future = producer.send("train-realtime", value=data)

# .get() 파라미터
future.get(timeout=10)
#          ↑
#          최대 몇 초 기다릴지 (초 단위)
#          전송이 완료되면 그 전에 바로 반환
#          timeout 안에 완료 못 하면 KafkaTimeoutError 발생
#          기본값: 없음 (무한 대기) ← 반드시 넣는 습관
```

```python
# 반환값: RecordMetadata 객체
record_metadata = future.get(timeout=10)

print(record_metadata.topic)      # "train-realtime"  ← 전송한 토픽
print(record_metadata.partition)  # 0                 ← 들어간 파티션 번호
print(record_metadata.offset)     # 42                ← 파티션 내 몇 번째 메시지인지
```

```
timeout 을 넣어야 하는 이유:
  넣지 않으면 → 브로커 장애 시 무한 대기
  프로그램 전체가 멈춰버림
  timeout=10 이면 10초 안에 안 되면 KafkaTimeoutError 발생
  → except 로 잡아서 처리 가능

offset 의 의미:
  파티션 안에서 메시지에 순서대로 붙는 번호 (0부터 시작)
  offset=42 → 이 파티션에 42번째로 들어간 메시지
  Consumer 에서 "어디까지 읽었는지" 추적하는 기준이 됨
```

```python
# 실전 패턴: 에러 잡기
from kafka.errors import KafkaTimeoutError

try:
    future = producer.send("train-realtime", value=data)
    meta   = future.get(timeout=10)
    print(f"✅ offset={meta.offset}")
except KafkaTimeoutError:     #Error는 [[Kafka_Error_Handling_Retry]] 참고 
    print("❌ 전송 타임아웃 (10초 초과)")
```

```text
future.get() 을 아예 쓰지 않으면:
  완전 비동기 → 가장 빠름
  but 에러 발생 여부를 즉시 알 수 없음
  → 고성능 파이프라인 or 에러를 나중에 콜백으로 처리할 때
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