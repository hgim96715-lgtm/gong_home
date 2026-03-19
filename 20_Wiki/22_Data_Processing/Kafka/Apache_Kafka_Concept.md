---
aliases:
  - Kafka
  - 카프카
  - Event Streaming
  - Broker
tags:
  - Spark
  - Kafka
related:
  - "[[00_Kafka_HomePage]]"
  - "[[Kafka_Topic_Partition_Offset]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[Kafka_Python_Producer]]"
  - "[[CS_TCP_IP]]"
---
# Apache_Kafka_Concept — 카프카란?

## 한 줄 요약

```
데이터가 흐르는 고속도로
실시간으로 대용량 데이터를 안정적으로 보내고 받는 분산 메시지 플랫폼
```

---

---

# ① Python 직접 저장 vs Kafka

```
왜 Python 으로 바로 DB 저장하지 않고 Kafka 를 쓰는가?
```

```
Python 직접 저장:
  API → Python → PostgreSQL

  문제:
  DB 가 느려지면 수집도 같이 느려짐
  소비자가 하나 (DB 만)
  INSERT 실패 시 데이터 영구 손실

Kafka 를 끼우면:
  API → Python(Producer) → Kafka → Spark → PostgreSQL
                                  → Airflow → 배치 분석
                                  → 미래 소비자 추가 가능

  버퍼 역할    수집 속도와 저장 속도를 분리
  다중 소비    같은 데이터를 여러 곳에서 동시에 읽기 가능
  메시지 보존  기본 7일 → Consumer 실패해도 재처리 가능
```

---

---

# ② Pub/Sub 모델

```
발행(Publish) / 구독(Subscribe)

보내는 쪽(Producer) 과 받는 쪽(Consumer) 이
서로를 몰라도 됨 (Decoupled)

TV 비유:
  방송국(Producer)이 채널(Topic)에 뉴스를 송출
  시청자(Consumer)는 자기가 원하는 채널을 구독해서 시청
  방송국은 시청자가 몇 명인지 몰라도 됨
  시청자는 방송국이 어디있는지 몰라도 됨
```

```
Producer → [ Topic ] → Consumer A
                    → Consumer B
                    → Consumer C

Producer 는 한 번만 발행
Consumer 는 각자 독립적으로 읽음
```

---

---

# ③ 핵심 구성 요소

|용어|설명|비유|
|---|---|---|
|**Topic**|데이터가 들어가는 카테고리|TV 채널|
|**Partition**|토픽을 여러 조각으로 쪼갠 것|차선 (많을수록 빠름)|
|**Producer**|데이터를 만들어 토픽에 보내는 주체|방송국|
|**Consumer**|토픽에서 데이터를 가져가는 주체|시청자|
|**Broker**|Kafka 서버 그 자체|송출탑|
|**Cluster**|여러 브로커가 모인 전체 시스템|방송국 전체|

---

---

# ④ KRaft — Zookeeper 없는 구조 (현재 표준)

```
과거: Kafka + Zookeeper (관리 프로그램 별도 필요)
현재: Kafka 3.3+ KRaft 모드 → Kafka 단독 실행 가능

우리 프로젝트 (apache/kafka:3.7.0):
  KRaft 모드 사용 → Zookeeper 컨테이너 불필요
  KAFKA_PROCESS_ROLES: broker,controller
  → 하나의 컨테이너가 브로커 + 컨트롤러 역할 동시 수행
```

---

---

# ⑤ 데이터 흐름

```
1. Producer 가 데이터를 Topic 에 Push (밀어넣기)
2. Broker 의 Partition 에 나뉘어 저장
3. Consumer 가 데이터를 Pull (당겨오기)

왜 Push 가 아니라 Pull 인가:
  Consumer 가 자기 처리 속도에 맞게 데이터를 가져감
  → Consumer 과부하 방지
  → Consumer 가 느려도 Kafka 에 데이터가 쌓여서 안전
```

---

---

# ⑥ Kafka 핵심 특징

```
높은 처리량 (High-throughput):
  초당 수백만 건 메시지 처리 가능
  디스크 순차 쓰기로 빠름

내결함성 (Fault-tolerant):
  데이터를 여러 브로커에 복제(Replication)
  일부 서버 고장나도 데이터 안전

영구 저장 (Event Persistence):
  메모리가 아닌 디스크에 저장 (기본 7일)
  Consumer 실패 → Offset 으로 위치 기억 → 재처리 가능
  과거 데이터 Replay 가능
```

---

---

# ⑦ 활용 사례

```
실시간 모니터링:
  병원 응급실 병상 현황 5분마다 수집
  → Kafka → Spark Streaming → 대시보드

로그 집계:
  여러 서버 로그를 Kafka 로 한곳에 모음

데이터 통합:
  서로 다른 시스템(DB / 앱 / API) 간 데이터 허브
```

---

---

# 한눈에 정리

```
Python 직접 → 소비자 하나 / 실패 시 손실 / 속도 종속
Kafka 사용  → 다중 소비 / 7일 보존 / 속도 분리

Producer → Topic(Partition) → Consumer
              ↑ Broker 에 저장
```

> 파티션 / 오프셋 상세 → [[Kafka_Topic_Partition_Offset]] Docker 환경 설정 → [[Docker_Host_vs_Internal_Network]]

---
---
# 소켓 vs Kafka

```
소켓 (Socket):
  1:1 연결 (클라이언트 ↔ 서버)
  연결이 끊기면 데이터 유실
  저장 없음 → 실시간 전송에 집중
  TCP 연결 기반

Kafka:
  1:N 연결 (Producer → 여러 Consumer)
  디스크에 저장 → 연결 끊겨도 데이터 보존
  Consumer 가 나중에 읽어도 됨 (Offset)
  소켓 위에서 동작 (Kafka 브로커 간 통신도 TCP 소켓)
```

```
언제 소켓:
  단순 1:1 실시간 통신
  채팅 / 게임 / 센서 직접 수신

언제 Kafka:
  여러 소비자가 같은 데이터 필요
  데이터 유실 없이 안정적으로 처리
  수집 속도 ≠ 처리 속도 인 경우
  → 데이터 파이프라인에는 Kafka
```

>[[CS_TCP_IP#소켓 (Socket)|소켓이란?]] 참고