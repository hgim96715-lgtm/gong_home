---
aliases:
  - Flink vs Spark 비교
  - 스트리밍 엔진 비교
tags:
  - Spark
  - PyFlink
related:
  - "[[Batch_vs_Stream]]"
  - "[[Flink_Introduction]]"
  - "[[00_Apache Flink_HomePage]]"
---
## 핵심 차이: 처리 모델 (Processing Model)

가장 근본적인 차이는 **"데이터를 어떻게 흐르게 하는가"** 에 있습니다. 

| 특징        | Apache Spark                                 | Apache Flink                                           |
| :-------- | :------------------------------------------- | :----------------------------------------------------- |
| **기본 철학** | **Batch-first**<br>(배치 엔진인데 스트리밍도 됨)         | **Streaming-first**<br>(스트리밍 엔진인데 배치도 됨)               |
| **처리 방식** | **Micro-batch**<br>데이터를 작은 덩어리로 모아서 처리함      | **Event-driven**<br>데이터가 들어오는 즉시(Record-by-record) 처리함 |
| **지연 시간** | **수백 밀리초 ~ 초 (Seconds)**<br>(높은 처리량, 적당한 지연) | **수 밀리초 (Sub-second)**<br>(극도로 짧은 지연 시간)               |

> **비유:**
> * **Spark:** 버스 배차 간격(0.5초)마다 승객을 태워서 출발함.
> * **Flink:** 승객이 오면 택시처럼 바로바로 출발함.

---
## 기능 및 성능 비교 (Features & Performance)

왜 실시간 처리에 Flink를 써야 하는지 보여주는 기술적 차이입니다. 

### ① 지연 시간 (Latency)

* **Spark:** 구조적으로 데이터를 모으는 시간(Micro-batch)이 필요해서 지연이 생길 수밖에 없습니다.
* **Flink:** **진정한 실시간(Native Streaming)** 이므로 금융 사기 탐지(Fraud Detection)처럼 0.1초가 급한 서비스에 유리합니다.

### ② 이벤트 시간 (Event Time)

* **Spark:** 기본적인 지원은 하지만, 늦게 도착한 데이터 처리가 복잡합니다.
* **Flink:** 워터마크(Watermark) 기능을 통해 **이벤트 발생 시간** 처리가 매우 강력하고 정교합니다.

### ③ 상태 관리 (State Management)

* **Spark:** 상태가 없는(Stateless) 작업에 유리합니다.
* **Flink:** 체크포인트(Checkpoint)를 통해 대규모 상태(Stateful)를 **정확히 한 번(Exactly-once)** 저장하고 복구하는 능력이 탁월합니다.

---
## 언제 무엇을 써야 할까? (Use Cases)

현업에서는 보통 두 기술을 적재적소에 섞어서 사용합니다. 

###  Spark를 써야 할 때 (Left Side)

1.  **대규모 배치 작업 (Batch ETL):** 어제 쌓인 100TB 로그 분석하기 
2.  **머신러닝 (ML Models):** `MLlib` 라이브러리가 훨씬 강력하고 성숙함 
3.  **Python 사용자:** `PySpark` 생태계가 압도적으로 큼 

###  Flink를 써야 할 때 (Right Side)

1.  **초저지연 분석 (Real-Time Analytics):** 실시간 주식 거래, 이상 탐지 
2.  **복잡한 이벤트 처리 (CEP):** "A 사건 발생 후 5분 안에 B 사건이 안 터지면 알람" 같은 패턴 감지 
3.  **대용량 스트리밍:** 초당 수백만 건의 이벤트를 쉼 없이 처리할 때 

---
##  생태계 요약 (Ecosystem)

| 구분        | Apache Spark            | Apache Flink           |
| :-------- | :---------------------- | :--------------------- |
| **주요 언어** | Python(강력), Scala, Java | Java(메인), Python(성장 중) |
| **ML 지원** | **Native (MLlib)**      | 제한적 (외부 라이브러리 의존)      |
| **커뮤니티**  | 거대함, 많은 레퍼런스            | 유럽/중국 강세 (알리바바 주도)     |