---
aliases:
  - Flink Checkpointing
  - Checkpoint vs Savepoint
  - State Backend
tags:
  - PyFlink
related:
  - "[[Flink_state]]"
  - "[[00_Apache Flink_HomePage]]"
---
## 한줄 요약 

**"장애가 나도 데이터를 잃어버리지 않게 '특정 지점(Snapshot)'을 사진 찍듯 저장하는 기술."**
Flink가 자랑하는 **강력한 신뢰성(Reliability)** 의 핵심이며, 서버가 터져도 마지막으로 저장된 지점부터 정확하게 복구할 수 있게 해줍니다.

---
## 왜 필요한가? (Why)

**문제점:**
- 스트리밍 데이터는 끝없이 들어옵니다. 만약 서버가 고장 나서 재부팅되면, 메모리에 있던 "지금까지의 합계"가 다 날아가서 0부터 다시 시작해야 합니다.

**해결책:**

- **Checkpointing:** 주기적으로 상태(State)와 데이터 위치(Offset)를 찰칵 찍어서(Snapshot) 안전한 곳(Disk/S3)에 보관합니다.
- 장애가 발생하면 가장 최근 체크포인트를 불러와서, 딱 그 시점부터 다시 데이터를 흘려보냅니다. 이를 통해 **Exactly-once(정확히 한 번 처리)** 를 보장합니다.

---
## 실전 맥락 (Practical Context)

데이터 엔지니어는 이 기능을 통해 **"데이터 유실 없는 파이프라인"**을 구축합니다.

### 1. Checkpointing (자동 저장)

- **특징:** Flink가 알아서 주기적으로 백그라운드에서 수행합니다. 기본값은 꺼져있으므로 설정해줘야 합니다.
- **목적:** **장애 복구(Fault Tolerance)**. 갑자기 죽었을 때 되살리기 위함.

### 2. Savepoints (수동 저장)

- **특징:** 사용자가 원할 때 직접 트리거(Trigger)합니다.
- **목적:** **계획된 작업**. 코드를 수정해서 배포하거나(Upgrade), 장비를 교체하거나, A/B 테스트를 위해 잠시 멈출 때 사용합니다.

### 3. State Backends (저장소 위치)

- **역할:** "State를 평소에는 어디에 두고, 체크포인트 할 땐 어디에 저장할래?"를 결정합니다.
- **종류:**
    - **Memory/Heap:** 빠르지만 데이터가 크면 메모리 터짐.
    - **RocksDB:** 디스크를 사용하므로 대용량 State 저장 가능

---
## 상세 분석: 동작 원리 (The "How")

### ① Checkpoint Barrier (핵심 메커니즘)

Flink는 데이터 흐름 사이에 **'Barrier(장벽)'** 라는 특수 신호를 끼워 넣습니다.

1. 소스(Source)에서 `Barrier N`을 보냅니다.
2. 이 장벽이 연산자(Operator)에 도착하면, 연산자는 "아, 여기까지가 N번 체크포인트구나"라고 인식합니다.
3. 연산자는 하던 일을 잠시 멈추고(혹은 비동기로) 자기 상태를 **State Backend**에 저장합니다.
4. 저장이 끝나면 `Ack`(확인) 신호를 마스터(JobManager)에게 보냅니다.


### ② Aligned vs Unaligned Checkpoint

원래는 모든 파이프라인의 속도를 맞춰야(Alignment) 하지만, 데이터가 너무 많이 밀리면(Backpressure) 체크포인트도 늦어집니다. 이를 해결하기 위해 두 가지 방식이 있습니다.

- **Aligned Checkpoint (기본):** 모든 입력 채널의 Barrier가 도착할 때까지 기다렸다가 스냅샷을 찍습니다. (줄 맞춰서 사진 찍기)
- **Unaligned Checkpoint (고속):**
    
    - 기다리지 않습니다(Eliminates alignment phase).
    - Barrier를 받자마자 **"처리 안 되고 버퍼에 떠 있는 데이터(in-flight data)"까지 싹 다 포함**해서 저장해버립니다.
    - **장점:** 데이터가 폭주해서 밀리는 상황(Backpressure)에서도 체크포인트를 빨리 찍을 수 있습니다.

---
## Code Core Points (아키텍처)

**JobManager와 TaskManager의 협동**

1. **JobManager (Master):** "자, 체크포인트 찍자!" 하고 `Trigger Checkpoint` 명령을 내립니다.    
2. **TaskManager (Worker):** 명령을 받으면 로컬 상태를 스냅샷 떠서 영구 저장소(Snapshot Store)에 올립니다.
3. **Ack:** 저장이 다 끝나면 "저 다 했어요"라고 JobManager에게 보고합니다.
4. 모든 TaskManager가 보고하면 체크포인트 완료! .

---
## Common Beginner Misconceptions (초보자의 실수)

### ① "배치(Batch) 처리할 때도 체크포인트 켜야 하나요?"

- **오해:** 배치도 State를 쓰니까 켜야 한다고 생각함.
- **현실:** 배치 모드(Batch Mode)에서는 상태(State)를 쓰긴 하지만, 스트리밍처럼 **장애 복구(Fault Recovery)나 체크포인팅 용도로 쓰지는 않습니다**. 
- 배치는 죽으면 그냥 처음부터 다시 돌리는 게 일반적이기 때문입니다.

### ② "Savepoint는 자동으로 안 생기나요?"

- **현실:** Savepoint는 **100% 수동(Manual)** 입니다. 배포 전에 개발자가 직접 명령어를 쳐서 만들어야 합니다. "체크포인트 있으니까 괜찮겠지?" 하고 그냥 끄면, 코드가 바뀌었을 때 복구가 안 될 수도 있습니다. (코드 구조가 바뀌면 체크포인트 호환이 안 될 수 있음)