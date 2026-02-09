---
aliases: [Flink Home, Flink MOC, PyFlink Study]
tags: [ApacheFlink, PyFlink, DataEngineering, Kafka, StudyMap]
cssclass: dashboard
---

# 🐿️ Apache Flink & PyFlink: The Event-Driven Journey

> [!QUOTE] 핵심 철학
> **"Spark가 '배치'를 잘게 쪼개서 스트리밍 흉내를 낸다면, Flink는 태생부터 '스트리밍'이다."**
> *진정한 Real-time Stream Processing과 저지연(Low Latency) 아키텍처를 향한 여정.*

---

## 📂 Quick Access (바로가기)

| 항목 | 경로 / 링크 | 비고 |
| :--- | :--- | :--- |
| **Project Root** | `/Users/gong/gong_study_de/apache-flink/playground/src` | 로컬 작업 공간 |
| **Source Code** | [📂 Playground 폴더 열기](file:///Users/gong/gong_study_de/apache-flink/playground/src/) | 실제 파이썬 코드 |
| **Diagram** | [[01_Apache Flink_Flow]] | 데이터 흐름도 |
| **Official Docs** | [PyFlink API Reference](https://nightlies.apache.org/flink/flink-docs-stable/api/python/_modules/pyflink/datastream/functions.html) | 공식 매뉴얼 |

---
## 1. Concept (이론 & 개념)

*Flink를 지탱하는 핵심 사상과 아키텍처 이해하기.*

- [[Flink_Introduction]] : **Flink란 무엇인가?** (Stateful Computations over Data Streams)
- [[Batch_vs_Stream]] : **Batch vs Streaming** (Bounded vs Unbounded 데이터의 차이)
- [[Flink_vs_Spark]] : **Flink vs Spark** (마이크로 배치 vs 네이티브 스트리밍)
- [[Flink_Architecture_Overview]] : **Architecture** (JobManager와 TaskManager의 역할 분담)
- [[Flink_Execution_Models]] : **실행 모드** (Session Mode vs Application Mode)
- [[Flink_Cluster_Deployment_Modes]] : **배포 전략** (Native Kubernetes, YARN, Standalone)


## 2. Environment (환경 구축)

*로컬(Mac)과 Docker를 활용한 격리된 개발 환경.*

- [[Flink_Docker_Setup(PyFlink)]] : **Docker Compose Setup** (이미지 빌드, `docker-compose.yml` 설정)


## 3. PyFlink Coding (코드 & 문법)

*DataStream API를 활용한 파이프라인 개발의 기초.*

- [[PyFlink_Import_Analysis]] : **Import 모음+분석** (TypeInfo, Source, Environment 등 필수 모듈)
- [[PyFlink_코드 해부_common ⭐️]] : **코드 뜯어보기** (Word Count 예제로 익히는 Flink 5단계 공식)

## 🔗 4. Kafka Integration (실전 연동) ⭐️
*정적 데이터(`fromData`)를 넘어선 리얼타임 파이프라인의 시작.*

> [!check] **Why Kafka?**
> PyFlink 학습 중 메모리 데이터(`fromData`)만으로는 실무적인 에러 핸들링과 스트리밍 특성을 배우기에 한계가 있음. **"진짜 데이터 흐름"**을 만들기 위해 Kafka를 도입함.

- [[PyFlink_Kafka_Docker_Setup]] : **Kafka 환경 구축** (docker-compose 설정, 네트워크 격리 해결)
- [[PyFlink_Kafka_Source]] : **Source 변경** (`fromData` → `KafkaSource` 코드 마이그레이션)
- [[PyFlink + Kafka 연동 완벽 가이드 ⭐️]] : **🔥 실행 & 트러블슈팅 매뉴얼** (무조건 성공하는 실행법)
- [[PyFlink_Kafka_Sink_Guide]] : **Sink 확장** (`sink_to` vs `add_sink`, 데이터 내보내기)
- [[PyFlink_Kafka_코드해부_Common ⭐️]] : **Import & 문법 완전 분석**
    1. **Environment 설정** (JAR 로딩)
    2. **Source (Input)** (KafkaSource 설정)
    3. **Transformation (Logic)** (TypeInfo, Map)
    4. **Sink (Output)** (KafkaSink 설정)
    5. **Execute** (Job 제출)

## ⚙️ 5. Operations (운영 & 모니터링)
*제출된 Job의 상태를 확인하고 관리하는 법.*

- [[Flink_Dashboard_Analysis]] : **Web UI 분석** (Job Graph, Checkpoints, Backpressure 확인법)
---


