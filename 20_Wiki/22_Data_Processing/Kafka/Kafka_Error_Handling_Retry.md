---
aliases:
  - 카프카 에러 처리
  - NoBrokersAvailable
  - KafkaException
  - 연결 실패 복구
  - Kafka retry
  - kafka try except
tags:
  - Kafka
related:
  - "[[00_Kafka_HomePage]]"
  - "[[Kafka_Python_Producer]]"
  - "[[Kafka_Python_Serialization]]"
  - "[[Kafka_Python_Consumer]]"
---

# Kafka_Error_Handling_Retry

## 개념 한 줄 요약

> **Kafka 는 네트워크 위에서 동작하기 때문에 언제든 연결이 끊길 수 있다.** 에러를 잡고, 기다렸다가, 다시 시도하는 로직이 없으면 파이프라인이 죽는다.

---

---

# 발생하는 에러 종류

|에러|언제 발생|원인|
|---|---|---|
|`NoBrokersAvailable`|Producer/Consumer 생성 시|Kafka 미실행, 주소 틀림, 컨테이너 아직 켜지는 중|
|`KafkaTimeoutError`|`send().get(timeout=10)`|토픽 없음, 브로커 응답 지연|
|`KafkaException`|전송/수신 중|브로커 연결 끊김, 파티션 리더 변경|
|`CommitFailedError`|`consumer.commit()`|리밸런싱 발생 중|

```python
from kafka.errors import (
    NoBrokersAvailable,
    KafkaTimeoutError,
    KafkaException,
    CommitFailedError,
)
```

---

---

# ① 기본 try / except 구조

```python
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable, KafkaException
import json

try:
    producer = KafkaProducer(
        bootstrap_servers="kafka:9092",
        value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
    )
except NoBrokersAvailable:
    # Kafka 가 아직 안 켜졌거나 주소가 틀림
    print("Kafka 연결 실패. 주소 및 컨테이너 상태를 확인하세요.")

# 전송 에러
try:
    future = producer.send("train-realtime", value={"trn_no": "00051"})
    future.get(timeout=10)   # 전송 결과 확인
except KafkaTimeoutError:
    print("전송 타임아웃. 토픽이 존재하는지 확인하세요.")
except KafkaException as e:
    print(f"Kafka 에러: {e}")
```

---

---

# ② 재시도 — 고정 간격 (Fixed Retry)

```python
import time
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable

MAX_RETRY  = 10   # 최대 시도 횟수
WAIT_SEC   = 3    # 매번 고정 대기 시간 (초)

producer = None
for attempt in range(1, MAX_RETRY + 1):
    try:
        producer = KafkaProducer(bootstrap_servers="kafka:9092")
        print(f"✅ 연결 성공 ({attempt}회 시도)")
        break
    except NoBrokersAvailable:
        print(f"⏳ 연결 실패 ({attempt}/{MAX_RETRY}) — {WAIT_SEC}초 후 재시도")
        time.sleep(WAIT_SEC)

if producer is None:
    raise RuntimeError("❌ Kafka 연결 최종 실패")
```

```
고정 간격 재시도는 단순하지만
서버가 과부하 상태일 때 모든 클라이언트가 동시에 재시도
→ 서버 부하를 더 키울 수 있음
→ 간격이 길어지는 백오프 방식이 더 안전
```

---

---

# ③ 지수 백오프 (Exponential Backoff) — 권장 ⭐️

```
재시도할수록 대기 시간을 점점 늘리는 방식

1회 실패 → 1초 대기
2회 실패 → 2초 대기
3회 실패 → 4초 대기
4회 실패 → 8초 대기
...
최대 60초 상한선 설정 (무한정 늘어나지 않게)

서버가 회복될 시간을 주면서 부하도 분산시킴
```

```python
import time
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable

MAX_RETRY  = 10
MAX_WAIT   = 60   # 최대 대기 시간 상한선 (초)

producer  = None
wait_time = 1     # 처음 대기 시간

for attempt in range(1, MAX_RETRY + 1):
    try:
        producer = KafkaProducer(bootstrap_servers="kafka:9092")
        print(f"✅ 연결 성공 ({attempt}회 시도)")
        break
    except NoBrokersAvailable:
        print(f"⏳ 연결 실패 ({attempt}/{MAX_RETRY}) — {wait_time}초 후 재시도")
        time.sleep(wait_time)
        wait_time = min(wait_time * 2, MAX_WAIT)  # 2배씩 증가, 상한선 60초

if producer is None:
    raise RuntimeError("❌ Kafka 연결 최종 실패")
```

---

---

# ④ 좀비 패턴 — Producer 가 죽지 않는 루프

```
"좀비 패턴" = 에러가 나도 죽지 않고 계속 살아있는 루프

데이터 파이프라인에서 Producer 가 갑자기 죽으면
  Kafka 에 데이터 공백이 생김
  Consumer 쪽도 영향

좀비처럼 에러를 먹고도 계속 살아서 전송을 이어가는 구조가 필요
```

```python
import time
import json
from datetime import datetime
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable, KafkaException

def create_producer(max_retry=10):
    """연결 재시도 + 지수 백오프로 Producer 생성"""
    wait_time = 1
    for attempt in range(1, max_retry + 1):
        try:
            producer = KafkaProducer(
                bootstrap_servers="kafka:9092",
                value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
                acks="all",
                retries=3,
            )
            print(f"✅ Producer 연결 성공 ({attempt}회 시도)")
            return producer
        except NoBrokersAvailable:
            print(f"⏳ 연결 실패 ({attempt}/{max_retry}) — {wait_time}초 후 재시도")
            time.sleep(wait_time)
            wait_time = min(wait_time * 2, 60)
    raise RuntimeError("❌ Producer 생성 최종 실패")


def send_with_retry(producer, topic, data, max_retry=3):
    """단건 전송 + 실패 시 재시도"""
    for attempt in range(1, max_retry + 1):
        try:
            future = producer.send(topic, value=data)
            meta   = future.get(timeout=10)
            print(f"  ✅ 전송 완료 partition={meta.partition} offset={meta.offset}")
            return True
        except KafkaException as e:
            print(f"  ❌ 전송 실패 ({attempt}/{max_retry}): {e}")
            time.sleep(2 ** attempt)   # 1회→2초, 2회→4초, 3회→8초
    print(f"  🚨 [{topic}] 전송 최종 실패. 메시지 폐기: {data}")
    return False


# 메인 루프 — 좀비처럼 살아있기
producer = create_producer()

while True:
    try:
        data = {
            "trn_no"      : "00051",
            "collected_at": datetime.now().isoformat(),
        }
        send_with_retry(producer, "train-realtime", data)
        time.sleep(60)

    except KeyboardInterrupt:
        # Ctrl+C 는 의도적인 종료 → 정상 종료
        print("\n🛑 종료 신호 수신. Producer 닫는 중...")
        producer.flush()
        producer.close()
        break

    except Exception as e:
        # 예상 못한 에러 → 죽지 말고 로그 남기고 계속
        print(f"⚠️ 예상치 못한 에러 발생: {e}")
        print("   5초 후 루프 재시작...")
        time.sleep(5)
        # Producer 연결이 끊겼을 수 있으니 재생성 시도
        try:
            producer = create_producer()
        except RuntimeError:
            print("🚨 Producer 재연결 실패. 종료.")
            break
```

---

---

# ⑤ Consumer 에러 처리

```python
from kafka import KafkaConsumer
from kafka.errors import NoBrokersAvailable, CommitFailedError
import json, time

def create_consumer():
    wait_time = 1
    for attempt in range(1, 11):
        try:
            return KafkaConsumer(
                "train-realtime",
                bootstrap_servers="kafka:9092",
                group_id="train-consumer-group",
                auto_offset_reset="latest",
                enable_auto_commit=False,   # 수동 commit 으로 안전하게
                value_deserializer=lambda x: json.loads(x.decode("utf-8")),
            )
        except NoBrokersAvailable:
            print(f"⏳ Consumer 연결 대기 ({attempt}/10) — {wait_time}초")
            time.sleep(wait_time)
            wait_time = min(wait_time * 2, 60)
    raise RuntimeError("Consumer 연결 최종 실패")


consumer = create_consumer()

try:
    for message in consumer:
        try:
            data = message.value
            # 처리 로직
            process(data)

            # 처리 완료 후 수동 commit
            try:
                consumer.commit()
            except CommitFailedError:
                # 리밸런싱 중 commit 실패 → 다음 루프에서 재처리됨
                print("⚠️ Commit 실패 (리밸런싱 중). 다음 루프에서 재처리됨")

        except Exception as e:
            # 메시지 처리 중 에러 → 이 메시지만 건너뛰고 계속
            print(f"⚠️ 메시지 처리 에러 (건너뜀): {e}")
            print(f"   문제 메시지: {message.value}")
            continue

except KeyboardInterrupt:
    print("\n🛑 종료")

finally:
    consumer.close()
```

---

---

# 에러 처리 전략 정리

```
에러 종류에 따라 전략이 달라짐

연결 에러 (NoBrokersAvailable)
  → 지수 백오프로 재시도
  → 최대 횟수 초과 시 RuntimeError 발생 후 종료

전송 에러 (KafkaException, KafkaTimeoutError)
  → 단건 재시도 (최대 3회)
  → 그래도 실패하면 로그 남기고 해당 메시지 폐기 or DLQ

메시지 처리 에러 (Consumer 로직 에러)
  → continue 로 해당 메시지 건너뜀
  → 전체 루프는 계속 유지

예상치 못한 에러 (Exception)
  → 루프 유지 (좀비 패턴)
  → Producer/Consumer 재생성 시도
  → Ctrl+C (KeyboardInterrupt) 는 의도적 종료로 별도 처리
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|재시도 없이 바로 죽음|`except` 없이 `NoBrokersAvailable` 미처리|`try/except NoBrokersAvailable` 추가|
|재시도가 서버 부하를 더 키움|고정 간격 동시 재시도|지수 백오프로 변경|
|처리 중 에러나면 루프 전체 종료|`while True` 안에 `try/except` 없음|루프 안에 개별 `try/except` 추가|
|Commit 실패로 데이터 중복 처리|리밸런싱 중 `CommitFailedError`|`CommitFailedError` 잡고 다음 루프에서 재처리|
|`KeyboardInterrupt` 도 재시도 루프에 잡힘|`except Exception` 이 너무 광범위|`except KeyboardInterrupt` 를 먼저, `except Exception` 을 나중에|