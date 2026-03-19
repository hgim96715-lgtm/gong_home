---
aliases:
  - Spark기초
  - 스파크개념
  - MapReduce_vs_Spark
  - RDD
  - In-Memory
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[CS_RAM_Memory]]"
  - "[[Apache_Kafka_Concept]]"
  - "[[Kafka_Python_Consumer]]"
---


# Spark_Concept_Evolution — Apache Spark 개념

## 한 줄 요약

```
Hadoop 의 느린 디스크 I/O 를 해결하기 위해
데이터를 메모리(RAM) 에 올려서 최대 100배 빠르게 처리하는
분산 컴퓨팅 엔진
```

---

---

# ① 왜 Spark 가 나왔는가 — Hadoop 의 한계

## Hadoop MapReduce (과거)

```
처리 방식:
  데이터 읽기 → 처리 → 디스크에 쓰기
  다음 단계 → 디스크에서 다시 읽기 → 처리 → 디스크에 쓰기

문제:
  매 단계마다 디스크 I/O 발생
  머신러닝처럼 데이터를 100번 반복 읽는 작업
  → 디스크 속도 병목 → 너무 느림
```

## Apache Spark (현재)

```
처리 방식:
  데이터를 RAM 에 올려두고 거기서 계속 작업
  → 디스크를 거의 안 씀

결과:
  Hadoop MapReduce 대비 최대 100배 빠름
  배치 / 스트리밍 / ML / 그래프 모두 하나의 엔진에서 처리
```

---

---

# ② 아키텍처 — "시키는 놈 vs 일하는 놈"

```
Driver ──→ Cluster Manager ──→ Executor (Worker)
(총사령관)    (자원 관리자)        (일꾼)
```

## Driver Program — 총사령관

```
역할:
  사용자 코드(PySpark) 를 해석
  작업 계획(DAG) 을 짜서 Executor 에게 명령

SparkContext:
  Driver 안에 있는 핵심 객체
  클러스터와 통신하는 접속 창구
  SparkSession = SparkContext 의 현대적 래퍼 (PySpark 에서 사용)
```

## Cluster Manager — 자원 관리자

```
역할:
  Driver 가 "CPU 100개, 메모리 200GB 필요해" 요청하면
  워커 노드에서 자원 할당해주는 중개자

종류:
  Standalone  Spark 자체 내장 클러스터 매니저 ← 우리 프로젝트
  YARN        Hadoop 생태계
  Kubernetes  컨테이너 기반
  Mesos       범용 클러스터
```

## Executor — 일꾼

```
역할:
  실제 데이터를 처리하는 Worker Node 의 프로세스
  Driver 가 내려보낸 Task 를 실행

Task:
  Driver 가 쪼갠 작업 단위 (Partition 하나 = Task 하나)

Cache:
  처리 중인 데이터를 메모리에 저장
  → 반복 작업 시 디스크 안 읽어도 됨 (속도의 비결)
```

---

---

# ③ Spark 5가지 컴포넌트

|컴포넌트|역할|사용 빈도|
|---|---|---|
|**Spark Core**|메모리 관리 / 작업 스케줄링 / 기반 엔진|내부 동작|
|**Spark SQL / DataFrame**|SQL 처럼 데이터 처리|⭐️ DE 90%|
|**Spark Streaming**|실시간 데이터 처리 (Kafka 연동)|⭐️ 이 프로젝트|
|MLlib|머신러닝 라이브러리|DS 영역|
|GraphX|그래프 데이터 분석|특수 용도|

```
데이터 엔지니어:
  Spark SQL / DataFrame + Spark Streaming 이 핵심
  나머지는 알아두는 수준으로 충분
```

---

---

# ④ 왜 PySpark 인가

```
Spark 는 원래 Scala 로 만들어짐
Python 바인딩이 PySpark

PySpark 를 쓰는 이유:
  문법이 Scala / Java 보다 훨씬 직관적
  Pandas / NumPy / Scikit-learn 과 연동 쉬움
  현업에서 가장 많이 쓰는 스킬셋

성능 차이:
  Scala > Java > PySpark (속도)
  BUT 대부분의 작업에서 체감 차이 없음
  → 생산성 + 생태계 고려하면 PySpark 선택이 합리적
```

---

---

# ⑤ Spark vs Hadoop — 오해 정리

## "Spark 가 Hadoop 을 완전히 대체한다?"

```
연산(Computing):
  MapReduce → Spark 로 대체 ✅
  훨씬 빠르고 편함

저장(Storage):
  Spark 는 저장소가 없음
  데이터는 여전히 HDFS / S3 / GCS 에 저장
  Spark 는 거기서 읽어서 처리하는 엔진

결론:
  저장: HDFS / S3   (Hadoop 역할 유지)
  연산: Spark       (MapReduce 대체)
```

## "RAM 이 부족하면 Spark 못 쓴다?"

```
메모리 꽉 차면:
  자동으로 디스크 임시 사용 (Spill to Disk)
  속도가 느려질 뿐, 작업 자체는 가능

최적 성능을 위해 메모리 충분히 주는 것이 좋지만
필수 조건은 아님
```

---

---

# ⑥ KafkaConsumer vs Spark Structured Streaming

## KafkaConsumer (kafka-python)


```python
from kafka import KafkaConsumer

consumer = KafkaConsumer("topic", bootstrap_servers="kafka:9092")
for msg in consumer:
    data = json.loads(msg.value)
    insert_to_postgres(data)   # 건별 INSERT
```

```
특징:
  메시지 하나씩 받아서 바로 처리
  Python 코드로 완결 — 별도 프레임워크 없음
  건별 INSERT → 소량 데이터에 적합
  단순하고 직관적
```

## Spark Structured Streaming


```python
df = spark.readStream \
    .format("kafka") \
    .option("subscribe", "topic") \
    .load()

df.writeStream \
    .foreachBatch(write_to_postgres) \
    .trigger(processingTime="30 seconds") \
    .start()
```

```
특징:
  마이크로 배치 처리 — N초마다 묶어서 처리
  분산 처리 — 대량 데이터도 병렬 처리
  DataFrame API — 집계 / 변환 / 조인 SQL 수준
  checkpointLocation — 장애 복구 자동 관리
```

## 비교표

|항목|KafkaConsumer|Spark Structured Streaming|
|---|---|---|
|처리 방식|건별 실시간|마이크로 배치 (N초마다)|
|처리량|소량 적합|대량 적합|
|변환/집계|Python 코드 직접|DataFrame API|
|장애 복구|직접 구현|checkpoint 자동|
|복잡도|단순|복잡|
|적합한 상황|단순 적재 / 소량|대량 / 집계 / 확장 필요|

>[[Kafka_Python_Consumer]]