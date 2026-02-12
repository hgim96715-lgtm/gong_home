---
aliases:
  - 카프카_토픽_파티션_오프셋
  - Partition
  - Offset
  - Kafka_Ordering
  - Topic
tags:
  - Kafka
related:
  - "[[Apache_Kafka_Concept]]"
---
## Concept Summary

**"고속도로(Topic)는 여러 개의 차선(Partition)으로 나뉘어 있고, 각 차량은 고유 번호판(Offset)을 달고 달린다."**

- **Topic:** 데이터가 들어가는 주제(폴더명).
- **Partition:** 토픽을 쪼개놓은 **물리적인 공간** (병렬 처리의 핵심).
- **Offset:** 파티션 내에서 데이터의 **순서 번호** (주소).

---
## 왜 필요한가? (Why)

### 1. 병렬 처리 (Parallelism)

- **문제:** 토픽이 하나인데 파티션도 하나라면? Producer가 아무리 빨리 보내도 Consumer 하나가 끙끙대며 처리해야 합니다. (1차선 도로 정체)
- **해결:** 파티션을 3개로 늘리면, Consumer 3명이 붙어서 동시에 처리할 수 있습니다. (3차선 고속도로 뻥 뚫림)

### 2. 순서 보장 (Ordering)

- **문제:** 분산 시스템에서 전역적인(Global) 순서 보장은 불가능에 가깝습니다.
- **해결:** Kafka는 **"하나의 파티션 안에서는 무조건 순서를 지킨다"** 는 약속을 합니다.

### 3. 데이터 추적 (Tracking)

- **문제:** Consumer가 죽었다 살아나면 어디까지 읽었는지 어떻게 알까?
- **해결:** **Offset(번호표)** 을 기억해두면 됩니다. "나 5번까지 읽었어"라고 기록해두면, 재시작 시 6번부터 읽으면 됩니다.

---
## Detailed Analysis: 구조 해부

### ① Topic (토픽)

- 데이터를 구분하는 **논리적인 이름**입니다. (예: `iot-topic`, `user-log`)
- 파일 시스템의 **폴더**와 같습니다.

### ② Partition (파티션) - ⭐️ 핵심

- 토픽을 **수평으로 쪼갠 단위**입니다.
- **특징:**
    - 한 번 늘리면 **줄일 수 없습니다.** (신중하게 결정!)
    - Kafka의 **성능(Throughput)** 은 파티션 수에 비례합니다.
    - Consumer Group의 Consumer 개수는 파티션 개수보다 많을 수 없습니다. (파티션 3개인데 Consumer 4개면, 1놈은 놉니다.)


### ③ Offset (오프셋)

- 파티션 내의 데이터 **고유 번호(ID)** 입니다. (0, 1, 2, 3...)
- **특징:**
    - **불변(Immutable):** 한 번 기록되면 절대 바뀌지 않습니다.
    - **파티션별 독립:** Partition 0의 오프셋 1번과 Partition 1의 오프셋 1번은 완전히 다른 데이터입니다.
    - 데이터가 삭제되어도 오프셋 번호는 줄어들거나 당겨지지 않고 계속 증가합니다.

---
## Practical Context: "순서 보장의 함정"

가장 많이 하는 오해입니다. **"Kafka는 순서를 보장한다"** 는 반은 맞고 반은 틀립니다.

> **"Kafka는 파티션 내(Within Partition)에서만 순서를 보장합니다."**

---
## Code Core Points (Python)

파이썬 코드에서 파티션을 나누는 핵심은 **`key`** 파라미터입니다.

```python
# 1. 키(Key) 없이 보내기 (Round-Robin)
# 데이터가 파티션 0, 1, 2에 골고루(랜덤하게) 뿌려짐 -> 순서 보장 X
producer.send('my-topic', value='data-1')

# 2. 키(Key) 지정해서 보내기 (Hashing) ⭐️
# 'seoul'이라는 키를 가진 데이터는 무조건 특정 파티션(예: 0번)으로만 감 -> 순서 보장 O
producer.send(
    'my-topic', 
    key=b'seoul',  # 키는 바이트로 변환해야 함
    value=b'25.0'
)
```


### 터미널에서 파티션 확인하기 (CLI)

```python
# 내 토픽이 몇 개의 파티션으로 되어 있는지 확인
kafka-topics.sh --describe --topic iot-topic --bootstrap-server kafka:9092
```

결과 예시: `PartitionCount: 3`이라고 나오면 3차선 도로입니다.
