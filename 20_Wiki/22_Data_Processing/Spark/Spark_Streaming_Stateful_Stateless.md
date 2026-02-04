---
aliases:
  - Stateful vs Stateless
  - 상태 기반 변환
  - State Store
  - 무상태 변환
tags:
  - Spark
  - Streaming
related:
  - "[[Spark_Key_Transformations]]"
  - "[[Spark_Streaming_Intro]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/kafka_stateful.py
---
## 개요: 기억력이 있는가, 없는가? 

스파크 스트리밍 변환은 **"과거의 데이터를 기억하느냐"** 에 따라 두 가지로 나뉩니다.

| 구분 | 영어 명칭 | 의미 | 비유 |
| :--- | :--- | :--- | :--- |
| **무상태** | **Stateless** | 들어온 데이터만 처리하고 잊어버림 | **건망증 심한 계산원** (방금 계산한 손님도 까먹음) |
| **상태 기반** | **Stateful** | 과거의 데이터를 기억(저장)해두고 누적해서 처리함 | **단골을 알아보는 주인** (외상 장부를 가지고 있음) |

---
##  Stateless Transformation (무상태 변환) 

각각의 데이터나 배치를 **독립적**으로 처리합니다. 이전 배치에서 무슨 일이 있었는지 전혀 신경 쓰지 않습니다.

* **특징**:
    * 역사(History)나 문맥(Context)을 고려하지 않음.
    * **처리 속도가 매우 빠르고** 확장이 쉽습니다.
    * 주로 `Append`나 `Update` 모드만 사용 가능합니다 (Complete 불가).
* **대표 함수**:
    * `select`, `filter`, `map`, `flatMap`, `explode`.
* **예시**:
    * "들어오는 단어를 소문자로 바꿔라" (이전 단어가 뭐였든 상관없음).

---
## Stateful Transformation (상태 기반 변환) 

데이터가 처리되는 동안 **상태(State)나 문맥(Context)** 을 유지하고 업데이트합니다.

* **특징**:
    * **State Store(상태 저장소)** 라는 메모리 공간에 중간 결과(집계, 카운트 등)를 저장합니다.
    * 데이터가 계속 쌓이면 메모리가 부족해지는 **OOM(Out Of Memory)** 문제가 발생할 수 있습니다.
* **대표 함수**:
    * `aggregations` (count, sum), `joining` (스트림 간 조인), `grouping`, `windowing`.
* **종류**:
    * **Managed**: 스파크가 알아서 관리해줌 (예: `groupBy().count()`).
    * **Unmanaged**: 개발자가 직접 제어함 (`mapGroupsWithState` 등).

---
## 동작 원리 시각화 (Total Revenue Example) 

.**PID(상품ID)별 총매출**을 구하는 과정입니다.
### ① 첫 번째 이벤트 (초기화)

* **Input**: `PID 1000`, `Amount 1,000`
* **State Store**: 비어있으므로 새로 기록. `{1000: 1,000}`.

### ② 두 번째 이벤트 (누적)

* **Input**: `PID 1000`, `Amount 1,000` (또 들어옴)
* **State Store**: 기존 1,000을 기억해내서 더함. `{1000: 2,000}`.
* **Input**: `PID 1001`, `Amount 500` (새 상품)
* **State Store**: 새로 기록. `{1001: 500}`.

### ③ 세 번째 이벤트 (상태 업데이트)

* **Input**: `PID 1001`, `Amount 500`
* **State Store**: 기존 500에 더함. `{1001: 1,000}`.

> **핵심**: 마이크로 배치(Micro Batch)가 끝난 뒤에도 **State Store**는 메모리에 남아서 다음 배치를 기다립니다.

---
## 요약 비교 (Cheat Sheet)

| 특징 | Stateless (무상태) | Stateful (상태 유지) |
| :--- | :--- | :--- |
| **기억력** | 없음 (현재만 중요) | 있음 (과거+현재) |
| **의존성** | 데이터 간 독립적 | 이전 데이터에 의존적 |
| **저장소** | 필요 없음 | **State Store** 필수 |
| **대표 예시** | `filter`, `map` | `count`, `sum`, `window` |
| **주의사항** | 로직이 단순함 | 메모리 관리(OOM) 조심  |

---
