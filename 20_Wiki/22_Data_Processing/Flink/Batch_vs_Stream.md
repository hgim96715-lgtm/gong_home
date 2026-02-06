---
aliases:
  - 배치와 스트리밍
  - Batch vs Stream
  - 실시간 처리 기초
tags:
  - Concept
  - BigData
  - Architecture
  - Flink
related:
  - "[[00_Apache Flink_HomePage]]"
  - "[[Flink_Introduction]]"
---
## 한 줄 요약 (Concept Summary)

> **"배치(Batch)는 '이미 끝난 어제 일'을 한꺼번에 처리하는 것이고, 
> 스트리밍(Stream)은 '지금 일어나고 있는 일'을 즉시 처리하는 것입니다."** 

---
## 상세 비교 (Comparison)

두 방식은 데이터를 바라보는 관점 자체가 완전히 다릅니다. 

| 특징 (Characteristic) | 배치 처리 (Batch) | 스트림 처리 (Stream) |
| :--- | :--- | :--- |
| **데이터 (Data)** | **유한함 (Finite)**<br>시작과 끝이 있는 완성된 데이터셋  | **무한함 (Infinite)**<br>끝없이 계속 들어오는 연속적인 데이터  |
| **처리 시점 (When)** | **데이터가 다 모인 후**<br>(After data is collected)  | **데이터가 도착하자마자**<br>(As data arrives)  |
| **지연 시간 (Latency)** | **높음 (High)**<br>분(Minute) ~ 시간(Hour) 단위  | **낮음 (Low)**<br>밀리초(ms) ~ 초(Second) 단위  |
| **시간 개념 (Semantics)** | 처리 시간 (Process Time)<br>작업이 돌아가는 시간이 기준  | **이벤트 시간 (Event Time)**<br>실제 사건이 발생한 시간이 기준  |
| **장애 복구 (Fault Tolerance)** | 실패하면 다시 돌리면 됨<br>(Retry failed jobs)  | 상태(State) 복구와 체크포인트가 필수<br>(Checkpoints needed)  |
| **대표 기술 (Systems)** | Hadoop, **Spark (기본)**, Presto  | **Apache Flink**, Kafka Streams, Storm  |

---
## 상황별 예시 (Scenarios)

###  배치 (Batch): "자정에 몰아서 숙제하기"

* **상황:** "어제 하루 동안 얼마나 팔렸지?" 
* **동작:** 24시간 동안의 판매 로그를 쌓아뒀다가, 밤 12시(Midnight)가 되면 한 번에 집계 프로그램을 돌립니다. 
* **용도:** 일간 리포트(Daily Report), 대규모 ETL 작업, 과거 데이터 분석 

### 스트리밍 (Stream): "그때그때 바로 반응하기"

* **상황:** "지금 의심스러운 결제가 발생했어! 바로 알려줘!" 
* **동작:** 결제가 일어나는 **그 순간(The moment it happens)** 바로 분석하고 대시보드를 업데이트하거나 알림을 보냅니다. 
* **용도:** 이상 탐지(Fraud Detection), 실시간 추천, 라이브 대시보드 

---
## Flink의 관점 (Why Flink?)

보통은 배치 엔진(Spark)과 스트리밍 엔진(Storm)을 따로 썼지만, **Flink는 이 둘을 통합했습니다.**

* Flink 입장에서는 **"배치(Batch)"도 "스트리밍의 특수한 경우(끝이 정해진 스트림)"** 일 뿐입니다.
* 그래서 Flink를 배우면 두 가지 처리를 하나의 엔진으로 모두 할 수 있습니다.