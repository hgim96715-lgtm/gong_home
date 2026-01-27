---
aliases:
  - RDD
  - Resilient_Distributed_Dataset
  - 스파크데이터구조
  - 불변성
  - Lineage
tags:
  - Spark
  - RDD
related:
  - "[[Spark_Architecture]]"
  - "[[Lazy_Evaluation]]"
  - "[[Node_vs_Partition]]"
  - "[[Transformations_vs_Actions]]"
---
## 개념 한 줄 요약

**"스파크에서 사용하는 '변경 불가능한(Immutable) 분산(Distributed) 데이터 모음(Dataset)'이자, 고장이 나도 스스로 복구하는 불사신."**

* **Resilient (회복력 있는):** 노드 하나가 고장 나서 데이터가 날아가도, 족보(Lineage)를 보고 다시 만들어냅니다.
* **Distributed (분산된):** 데이터가 한 곳에 있지 않고 여러 컴퓨터(Node)에 쪼개져 있습니다.
* **Dataset (데이터 모음):** 우리가 처리할 실제 데이터 덩어리입니다.

---
## RDD의 5가지 핵심 특징 (Key Characteristics)

RDD가 왜 강력한지 설명하는 5가지 특징입니다.

### ① 불변성 (Immutability) 

* **정의:** 한 번 만들어진 RDD는 **수정할 수 없습니다(Read-Only).** 
* **이유:** 데이터가 변경되지 않아야 여러 노드에서 동시에 처리해도 꼬이지 않기 때문입니다.
* **변경하고 싶다면?:** 기존 RDD를 수정하는 게 아니라, 변형(Transformation)을 통해 **새로운 RDD**를 만들어냅니다. 

### ② 분산 처리 (Distributed) 

* **정의:** RDD의 데이터는 클러스터 내의 여러 노드에 **파티션(Partition)** 단위로 쪼개져서 저장됩니다. 
* **효과:** 여러 노드가 각자 맡은 파티션을 동시에 처리하므로(Parallel Processing) 속도가 빠릅니다. 

### ③ 회복력 (Resilience & Fault Tolerance) 

* **정의:** 데이터가 유실되어도 **자동으로 복구**합니다. 
* **원리 (Lineage):** RDD는 데이터를 직접 저장하는 게 아니라, **"이 데이터는 A에서 B를 거쳐 C가 되었다"** 는 **족보(Lineage)** 를 기억합니다. 
* 만약 C가 날아가면? B를 다시 계산해서 C를 만들어냅니다. 

### ④ 게으른 연산 (Lazy Evaluation) 

* **정의:** 변환 작업(Transformation)을 지시해도 **즉시 실행하지 않습니다.** 
* **작동:** "기다려, 아직 아니야..." 하다가 실제로 결과를 내야 하는 **액션(Action, 예: `collect`, `save`)** 이 호출될 때 비로소 계산을 시작합니다. 
* **이유:** 전체 작업 흐름을 보고 최적의 경로를 찾기 위해서입니다.

### ⑤ 파티셔닝 (Partitioned) 

* **정의:** 거대한 데이터를 **논리적인 조각(Partition)** 으로 나눕니다. 
* **특징:** 스파크의 병렬 처리는 이 파티션 단위로 일어납니다.

---

## RDD vs DataFrame

요즘은 RDD를 직접 쓰기보다 **DataFrame**을 많이 씁니다. 
하지만 RDD는 여전히 스파크의 **엔진(Foundation)** 입니다. 

| 구분 | **RDD (Low-Level)** | **DataFrame (High-Level)** |
| :--- | :--- | :--- |
| **데이터 형태** | 아무 객체나 다 담음 (비정형) | 행(Row)과 열(Column)이 있는 테이블 (정형) |
| **최적화** | 사용자가 직접 최적화해야 함 | 스파크가 알아서 최적화함 (Catalyst Optimizer) |
| **비유** | **"어셈블리어 / C언어"** | **"SQL / 파이썬"** |
| **사용 시기** | 아주 정밀한 제어가 필요할 때 | 일반적인 데이터 분석의 99% |

---

###  팁

"요즘 실무에서는 `spark.read.csv()` 처럼 **DataFrame**을 주로 쓰지만, 그 DataFrame도 결국 껍질을 까보면 **RDD**로 이루어져 있어.
특히 **'불변성'** 과 **'게으른 연산'** 은 스파크가 왜 에러가 났는지, 왜 속도가 느린지 디버깅할 때 아주 중요한 단서가 되니까 꼭 기억해둬!"