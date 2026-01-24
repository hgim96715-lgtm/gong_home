---
aliases:
  - Spark
  - 스파크
  - Big Data
  - Distributed Computing
  - In-memory
tags:
  - Spark
  - BigData
  - Framework
related:
  - "[[00_Airflow_HomePage]]"
---
## 개념 한 줄 요약

**Apache Spark**는 빅데이터 워크로드를 위해 설계된 **오픈소스 분산 데이터 처리 프레임워크(Distributed Data Processing Framework)** 야.
여러 대의 컴퓨터(노드)에 작업을 나누어 뿌려서(Distributing tasks), 대용량 데이터를 아주 빠르고 효율적으로 처리할 수 있어.

---
##  Why is it needed? (압도적인 속도) 

과거의 대장주였던 **Hadoop MapReduce**보다 최대 **100배**나 빨라.

* **비결은 In-Memory:** 하둡은 계산할 때마다 하드디스크에 쓰고 읽었는데, Spark는 **메모리(RAM)** 안에서 데이터를 유지하며 처리해.
* **Disk I/O 감소:** 디스크를 덜 긁으니까 성능이 비약적으로 향상된 거지.
----
## Key Features (만능 도구) 

Spark 하나만 배우면 온갖 작업을 다 할 수 있어 (Versatile Processing Model).

1.  **Batch Processing:** 쌓여있는 대용량 데이터 한 방에 처리.
2.  **Stream Processing:** 실시간으로 들어오는 데이터 처리 (Kafka 연동 등).
3.  **SQL Support:** 복잡한 코드 대신 SQL로 데이터 조회 가능 (**Spark SQL**).
4.  **Machine Learning:** 머신러닝 라이브러리(**MLlib**) 내장.
5.  **Ease of Use:** Java, Scala, **Python**, R 등 다양한 언어 지원 (우리는 Python을 쓰는 **PySpark**를 주로 쓸 거야!).

---
##  Architecture (어떻게 돌아가?) 

Spark는 **대장(Driver)** 이 **일꾼(Executor)** 들에게 일을 시키는 구조야.

* **Driver Program:** 메인 프로그램이야. `SparkContext`를 만들어서 클러스터와 대화해.
* **Cluster Manager:** 자원 관리반장. "이 작업 하려면 CPU랑 메모리 얼마나 필요해?"를 조율해 (YARN, Mesos, Kubernetes 등).
* **Executor:** 실제 노드(Worker Node)에서 작업을 수행(Execute)하는 일꾼들이야.

---
##  Use Cases (실전 활용)
[[Apache_Spark_Setup]]
* **Data Processing (ETL):** 원본 데이터(Raw Data)를 분석하기 좋게 변환할 때.
* **Real-Time Analytics:** Kafka 같은 곳에서 실시간 스트림 데이터를 받아서 처리할 때.
* **Machine Learning Pipelines:** 대용량 데이터셋으로 모델을 학습시키고 배포할 때.