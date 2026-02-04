---
aliases:
  - Streaming Join
  - Static Join
  - Cassandra
  - 카산드라
  - CQL
tags:
  - Spark
  - Streaming
  - Cassandra
related:
  - "[[Spark_Streaming_Intro]]"
  - "[[Spark_DataFrame_Joins]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Streaming_Static_Join_Run_Guide]]"
linked:
  - https://github.com/apache/cassandra-spark-connector/blob/trunk/doc/reference.md
---
## 개요: Streaming to Static Join 

스트리밍 처리를 하다 보면 실시간으로 들어오는 데이터만으로는 정보가 부족할 때가 있습니다.
이때 **외부 데이터베이스(Static Data)** 에 있는 마스터 데이터를 조회해서 합치는 작업을 **Streaming-Static Join**이라고 합니다.

* **시나리오:**
    * **Stream:** 유저 로그 (`user_id`, `action`, `time`)
    * **Static:** 유저 정보 테이블 (`user_id`, `name`, `age`)
    * **Join 결과:** 누가 언제 무엇을 했는지 상세 정보 포함 (`Kim`, `click`, `12:00`)

---
##  Apache Cassandra란? 

대용량 정적 데이터를 빠르게 조회하기 위해 자주 사용되는 **NoSQL 데이터베이스**입니다.

* **특징**:
    * **분산 아키텍처:** 여러 노드에 데이터를 분산 저장하여 확장성(Scalability)이 뛰어납니다.
    * **고가용성 (High Availability):** 단일 장애 지점(SPOF)이 없어 서버 하나가 죽어도 데이터 유실이 없습니다.
    * **높은 처리량:** 쓰기(Write)와 읽기(Read) 성능이 매우 뛰어납니다.
    * **CQL (Cassandra Query Language):** SQL과 매우 유사한 문법을 사용하여 배우기 쉽습니다.

---
## 환경 구축 (Installation & Config) 

실습을 위해 Docker를 사용하여 카산드라를 띄웁니다.

**Docker Compose 설정 (예시)**:
* **이미지:** `bitnami/cassandra:4.0.11`
* **포트:** `9042` (기본 통신 포트) 
* **볼륨:** 데이터 영구 저장을 위한 매핑 (`./cassandra-persistence:/bitnami`) 

```yaml
cassandra:
  image: cassandra:4.1              # Apache Cassandra 4.1 이미지
  container_name: cassandra         # 컨테이너 이름 (docker ps 식별용)

  ports:
    - "9042:9042"                   # CQL 포트 (Spark, cqlsh, 외부 접근)

  volumes:
    - cassandra-data:/var/lib/cassandra
      # Cassandra 데이터 디렉토리
      # 컨테이너 재시작/삭제 후에도 데이터 유지됨
      # 하단 volumes 섹션에 cassandra-data 정의 필요

  environment:
    - CASSANDRA_CLUSTER_NAME=SparkCluster
      # Cassandra 클러스터 이름

    - CASSANDRA_DC=datacenter1
      # 데이터센터 이름
      # Spark Cassandra Connector의 localDC 값과 반드시 일치해야 함

    - MAX_HEAP_SIZE=1G
      # JVM 최대 힙 메모리 크기

    - HEAP_NEWSIZE=200M
      # Young Generation 힙 크기 (GC 튜닝용)

  networks:
    - spark-net                     # Spark 컨테이너와 동일 네트워크

```

> 데이터 준비부터 실행까지 [[Spark_Streaming_Static_Join_Run_Guide]] 확인 
