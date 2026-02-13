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
---
## Concept Summary(한줄요약)

"데이터 배달부(Producer)와 문전박대 에러(NoBrokersAvailable)."

- **`KafkaProducer`**: 파이썬 데이터를 카프카 브로커(서버)로 쏘아 보내는 객체.
- **`NoBrokersAvailable`**: "야, 카프카 문 닫혀있는데?" 라며 연결 실패를 알리는 에러.

---
## 왜 필요한가? (Why)

- **Producer:** Flink나 Spark가 데이터를 처리하려면, 일단 누군가가 카프카에 데이터를 넣어줘야 합니다. 파이썬이 그 역할을 가장 쉽게 할 수 있습니다.
- **NoBrokersAvailable:** 도커(Docker) 환경에서는 Kafka가 켜지는 속도보다 파이썬이 실행되는 속도가 더 빠를 때가 많습니다. 이때 그냥 죽지 않고 **"기다렸다가 다시 연결"** 하기 위해 이 에러를 잡아야 합니다.

---
## Code Core Points (문법 해부)

### ① 필수 라이브러리 가져오기

```python
# pip install kafka-python 필요
from kafka import KafkaProducer
from kafka.errors import NoBrokersAvailable # 연결 실패 에러 처리용
```

- `kafka-python` 라이브러리가 설치되어 있어야 합니다. (`pip install kafka-python`)
- `NoBrokersAvailable`은 이름 그대로 **"브로커(카프카 서버)가 응답이 없다"** 는 뜻입니다.

### ②  프로듀서 생성 (옵션 설정) ⭐️

가장 중요한 부분입니다. 데이터를 보낼 **주소**와 **포장 방법**을 정합니다.

```python
producer = KafkaProducer(
    bootstrap_servers='kafka:9092', 
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)
```

- **`bootstrap_servers`**: "어디로 보낼까?"
	- 카프카 서버의 주소입니다. (로컬: `localhost:29092`, 도커 내부: `kafka:9092`)
	- 하나만 적어줘도 알아서 클러스터 전체 정보를 찾아냅니다(Bootstrap).
	- **내 맥북(터미널)에서 부를 때:** 도커 밖이니까 **"외부 연결용 문"** 인 `localhost:29092`
	- **도커 안(Spark/Flink)에서 부를 때:** 같은 도커 안이니까 **"내부 주소"** 인 `kafka:9092` 사용

> server를 뭘 할지 모르겠다면 [[Docker_Host_vs_Internal_Network]] 참고 

- **`value_serializer`**: "어떻게 포장할까?" (직렬화)
	- 카프카는 오직 **0과 1(Byte)** 만 알아듣습니다. 파이썬 딕셔너리(`{'a': 1}`)를 그대로 던지면 에러 납니다.
	- **공식:** `{python}lambda v: json.dumps(v).encode('utf-8')`
		- `v` (데이터)를 받아서>`json.dumps` (문자열로 바꾸고)>`.encode('utf-8')` (최종적으로 바이트로 변환)

### ③ 연결 재시도 패턴 (Retry Logic) 

"카프카 켜질 때까지 10번만 봐준다." 하는 국룰 코드입니다.

```python
import time

# 10번 시도한다 (기회는 10번)
for _ in range(10):
    try:
        # 연결 시도!
        producer = KafkaProducer(...)
        
        # 성공하면 바로 반복문 탈출 (성공!)
        break 
        
    except NoBrokersAvailable:
        # 실패하면 여기로 옴
        print("카프카 아직 자는 중... 3초 뒤에 다시 깨움.")
        time.sleep(3)
else:
    # for문이 10번 다 돌 때까지 break를 못 만났다? (= 10번 다 실패)
    raise RuntimeError("10번이나 깨웠는데 안 일어남. 시스템 종료.")
```

- **`for - else` 문법:** 파이썬의 특이한 문법입니다. 
	- `for`문이 중간에 `break` 없이 끝까지 다 돌면 `else` 블록이 실행됩니다. (즉, "완전 실패" 시 처리)

### ④ 데이터 전송 (`send`)

- **문법:** `send(토픽이름,데이터)`

```python
# send(토픽이름, 데이터)
producer.send('my-topic', value={'name': 'gong'})

# 큐에 쌓인 데이터를 강제로 밀어넣기 (확실하게 전송)
producer.flush()
```

- `send`는 비동기(Async)입니다. "보내!" 하고 바로 다음 줄로 넘어갑니다.
- 프로그램이 금방 꺼지면 데이터가 아직 출발도 안 했을 수 있으니, 마지막에 `flush()`로 "남은 거 싹 다 보내고 퇴근해!"라고 명령해야 합니다.

---
## Common Beginner Misconceptions (초보자 실수)

### ① "NoBrokersAvailable이 계속 떠요!"

- **원인 1:** 주소(`bootstrap_servers`)가 틀렸음. (localhost vs kafka 헷갈림)
- **원인 2:** 카프카 컨테이너가 죽었거나 아직 켜지는 중임.
- **원인 3:** Docker 네트워크가 서로 다름. (같은 `docker-compose` 안에 있어야 함)

### ② "전송했는데 카프카에 데이터가 없어요."

- **원인:** `value_serializer` 설정을 안 했거나, `flush()`를 안 하고 프로그램을 종료해버려서 전송 큐에만 쌓여있다가 증발한 경우입니다.

### ③ "JSON 말고 그냥 글자(String) 보내면 안 되나요?"

- 됩니다! 단, 그때는 `value_serializer`를 `lambda v: v.encode('utf-8')`로 바꿔야 합니다. (`json.dumps` 제거). 핵심은 **무조건 바이트(Byte)로 만들어야 한다**는 점입니다.