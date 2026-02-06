---
aliases:
  - Apache Flink란?
  - Flink 소개
  - Stratosphere
tags:
  - Flink
related:
  - "[[00_Apache Flink_HomePage]]"
---
# Apache Flink란 무엇인가?

## 1. 정의 (Definition)

> **"Data is no longer just stored, it's always in motion."** 

**Apache Flink**는 **"스트리밍 우선(Streaming-first)"** 으로 설계된 분산 데이터 처리 엔진입니다.
배치(Batch) 처리를 중심으로 하던 기존 하둡(Hadoop) 생태계와 달리,
Flink는 데이터가 생성되는 즉시 처리하는 **실시간 애플리케이션**을 위해 태어났습니다.

* **무한 데이터(Unbounded)와 유한 데이터(Bounded/Batch)** 를 모두 처리할 수 있습니다.
* **인메모리(In-memory)** 속도로 대규모 클러스터에서 초당 수백만 건의 이벤트를 처리합니다.
* Alibaba, Netflix, Uber 등 글로벌 테크 기업들이 사용 중입니다.

---
##  왜 Flink인가? (Key Features)

스파크(Spark)도 스트리밍이 되지만, Flink는 아래와 같은 **결정적인 차별점**이 있습니다.

### ① 진정한 저지연 (True Low-Latency)

* **Spark Streaming:** 데이터를 작은 배치(Micro-batch)로 모아서 처리하므로 지연 시간이 발생합니다.
* **Flink:** 들어오는 데이터를 모으지 않고 **즉시(Event-at-a-time)** 처리합니다.

### ② 강력한 상태 일관성 (State Consistency)

* 장애가 발생해도 데이터가 유실되거나 중복되지 않도록 **"정확히 한 번(Exactly-once)"** 처리를 보장합니다.
* 내부적으로 **상태(State)** 를 저장하고 관리하는 기능이 매우 강력합니다.

### ③ 이벤트 시간 처리 (Event Time)

* 네트워크 지연으로 데이터가 늦게 도착해도(Out-of-order), 데이터가 **"실제로 발생한 시간"** 을 기준으로 올바르게 정렬하고 집계할 수 있습니다.

![[스크린샷 2026-02-05 오후 4.56.08.png|600x300]]

>Flink의 4대 특징
>**Open-source (노랑):** Apache 재단의 검증된 오픈소스 프로젝트.
>**Data Handling (주황):** 배치(Bounded)와 스트리밍(Unbounded) 데이터를 모두 처리하는 하이브리드 엔진.
>**Core Features (빨강):** 장애 복구(Fault Tolerance), 상태 관리(Stateful), 이벤트 시간(Event Time) 완벽 지원.
>**Processing Speed (분홍):** 분산 환경에서 초당 수백만 건을 처리하는 압도적 성능.

---
## 역사 (History)

* **2009년:** 베를린 공대(TU Berlin) 등의 연구 프로젝트인 **'Stratosphere'** 로 시작됨.
* **2014년:** Apache 재단의 탑 레벨 프로젝트(Top-level Project)로 승격.
* **현재:** 배치 분석을 지배하던 Spark와 달리, **"Stateful Stream Processing"** 분야의 표준으로 자리 잡음.

---
## 추상화 계층 (API Abstraction Levels)

Flink는 사용자의 목적에 따라 4가지 레벨의 API를 제공합니다. 
위로 갈수록 쉽지만 제약이 있고, 아래로 갈수록 어렵지만 자유도가 높습니다.

1.  **Flink SQL (High Level)**: SQL 문법으로 실시간 데이터를 쿼리합니다.
2.  **Table API**: SQL과 비슷하지만 Python/Java 코드로 작성하는 관계형 API입니다.
3.  **DataStream API (Core)**: 스트림, 윈도우(Window) 등을 다루는 핵심 API입니다. (Spark의 RDD/DataFrame과 유사) .
4.  **Process Function (Low Level)**: 시간(Time), 상태(State), 워터마크를 직접 제어하는 가장 강력하고 복잡한 단계입니다.


![[스크린샷 2026-02-05 오후 5.01.55.png|600x300]]