---
aliases:
  - 스파크설치
  - Docker_Spark
  - PySpark설치
  - 로컬환경구축
  - Docker compose
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Docker_Compose_Setup]]"
  - "[[Spark_JDBC_PostgreSQL]]"
  - "[[Spark_Session_Context]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
  - "[[Kafka_Python_Consumer]]"
---
#  Spark_Installation_Local_Docker — Spark 환경 구성

## 한 줄 요약

```
로컬 테스트: pip install pyspark
실제 프로젝트: Docker Compose + JAR 파일 준비
```

---

---

# ① pip install pyspark — 로컬 테스트용

```bash
pip3 install pyspark==3.5.0
```

```
requirements.txt 에 추가:
  pyspark==3.5.0
```

```
⚠️ pip install pyspark 는 Spark 바이너리를 Python 패키지로 설치
   클러스터 없이 로컬에서 단독 실행 가능 (테스트 / 개발용)
   실제 분산 처리는 Docker Compose 로 클러스터 구성 필요
```

## JAVA_HOME 설정 — 필수

```
PySpark 는 JVM 위에서 동작 → Java 가 반드시 설치돼 있어야 함
```

```bash
# Java 설치 확인
java -version

# macOS — Java 설치
brew install openjdk@17

# JAVA_HOME 환경변수 설정 (.zshrc 에 추가)
export JAVA_HOME=$(/usr/libexec/java_home)
source ~/.zshrc

# 확인
echo $JAVA_HOME
# /Library/Java/JavaVirtualMachines/openjdk-17.jdk/Contents/Home
```

```
JAVA_HOME 없으면:
  JAVA_HOME is not set
  또는 py4j.protocol.Py4JError: ... 에러 발생
```

---

---

# ② Docker Compose — 실제 프로젝트

```
프로젝트에서 Spark 를 Docker 로 띄우는 방식
spark-master + spark-worker 컨테이너 구성
```

```yaml
# docker-compose.yml
services:

  spark-master:
    image: apache/spark:3.5.0
    container_name: hospital-spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "8083:8080"   # Spark Web UI
      - "7078:7077"   # Spark Master 포트
    volumes:
      - ./spark:/opt/spark/apps                                       # 코드 마운트
      - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
    networks:
      - hospital-network

  spark-worker:
    image: apache/spark:3.5.0
    container_name: hospital-spark-worker
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    environment:
      - SPARK_WORKER_MEMORY=1G
      - SPARK_WORKER_CORES=1
    volumes:
      - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
    depends_on:
      - spark-master
    networks:
      - hospital-network
```

```
⚠️ JAR 마운트는 반드시 파일 대 파일로
  - ./spark_drivers:/opt/spark/jars   ← ❌ 폴더째 마운트 금지
    → /opt/spark/jars 안의 Spark 기본 엔진 파일이 덮어씌워짐
    → Spark 자체가 망가짐

  - ./spark_drivers/파일.jar:/opt/spark/jars/파일.jar  ← ✅
```

---

---

# ③ spark_drivers/ — JAR 파일 준비

```
Kafka 연동 + PostgreSQL JDBC 에 필요한 JAR 5개
--packages 옵션으로 실행 시 Maven 에서 자동 다운로드도 되지만
매번 다운로드해서 느림 → 미리 받아두는 방식 권장
```

## JAR 5개 역할

|JAR 파일|역할|
|---|---|
|`spark-sql-kafka-0-10_2.12-3.5.0.jar`|**Kafka 연동 핵심**|
|`spark-token-provider-kafka-0-10_2.12-3.5.0.jar`|Kafka 토큰 인증 (의존성)|
|`kafka-clients-3.4.1.jar`|Kafka 클라이언트 (의존성)|
|`commons-pool2-2.11.1.jar`|커넥션 풀 (의존성)|
|`postgresql-42.7.1.jar`|PostgreSQL JDBC|

```
5개 모두 필요
하나라도 없으면 ClassNotFoundException 에러 발생
```

## 다운로드

```bash
# 프로젝트 루트에서 실행
mkdir -p spark_drivers
cd spark_drivers

curl -O https://repo1.maven.org/maven2/org/apache/spark/spark-sql-kafka-0-10_2.12/3.5.0/spark-sql-kafka-0-10_2.12-3.5.0.jar
curl -O https://repo1.maven.org/maven2/org/apache/spark/spark-token-provider-kafka-0-10_2.12/3.5.0/spark-token-provider-kafka-0-10_2.12-3.5.0.jar
curl -O https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/3.4.1/kafka-clients-3.4.1.jar
curl -O https://repo1.maven.org/maven2/org/apache/commons/commons-pool2/2.11.1/commons-pool2-2.11.1.jar
curl -O https://repo1.maven.org/maven2/org/postgresql/postgresql/42.7.1/postgresql-42.7.1.jar

# 확인
ls -l
# commons-pool2-2.11.1.jar
# kafka-clients-3.4.1.jar
# postgresql-42.7.1.jar
# spark-sql-kafka-0-10_2.12-3.5.0.jar
# spark-token-provider-kafka-0-10_2.12-3.5.0.jar
```

## 패키지 이름 해석

```
org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0
  org.apache.spark   조직 (Maven groupId)
  spark-sql-kafka-0-10  Kafka 0.10+ 용 커넥터
  _2.12              Scala 2.12 버전 (Spark 3.x 기준)
  :3.5.0             Spark 버전과 반드시 맞춰야 함 ⚠️
```

---

---
# ④ Consumer 실행 (STEP 4)

## docker cp — JAR 컨테이너에 복사

```bash
# container_name 은 docker-compose.yml 의 container_name 값과 동일
# hospital 프로젝트: hospital-spark-master
# train 프로젝트:    train-spark-master

docker cp spark_drivers/. hospital-spark-master:/opt/spark/jars/

# 복사 확인
docker exec -it hospital-spark-master ls /opt/spark/jars/ | grep -E "kafka|postgresql"
```

```
⚠️ docker compose down 후 재시작 시 매번 필요할 수 있음
   docker-compose.yml 볼륨 마운트가 정상이면 docker cp 불필요
   JAR 없을 때만 사용
```

## spark-submit 실행

```bash
# container_name 과 --master 안의 서비스명 주의
# docker exec -it [container_name] ...
# --master spark://[서비스명]:7077

docker exec -it hospital-spark-master \
  /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/spark-sql-kafka-0-10_2.12-3.5.0.jar,/opt/spark/jars/spark-token-provider-kafka-0-10_2.12-3.5.0.jar,/opt/spark/jars/kafka-clients-3.4.1.jar,/opt/spark/jars/commons-pool2-2.11.1.jar,/opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/consumer.py
```

```
헷갈리는 두 가지:

container_name (docker exec 에 쓰는 것):
  docker-compose.yml 의 container_name 값
  → hospital-spark-master / train-spark-master 등 프로젝트마다 다름

--master 안의 서비스명 (spark://서비스명:7077):
  docker-compose.yml 의 services: 키 이름
  → 보통 spark-master 로 고정
  → 컨테이너 내부 DNS 로 해석됨

예시:
  services:
    spark-master:                         ← --master spark://spark-master:7077
      container_name: hospital-spark-master  ← docker exec -it hospital-spark-master
```

---

---

# 폴더 구조

```
hospital-project/
├── docker-compose.yml
├── spark/
│   └── spark_consumer.py     ← Spark 코드
├── spark_drivers/            ← JAR 파일 (gitignore 추가 권장)
│   ├── spark-sql-kafka-0-10_2.12-3.5.0.jar
│   ├── spark-token-provider-kafka-0-10_2.12-3.5.0.jar
│   ├── kafka-clients-3.4.1.jar
│   ├── commons-pool2-2.11.1.jar
│   └── postgresql-42.7.1.jar
└── ...
```

```gitignore
# .gitignore 에 추가 (JAR 파일 용량 커서 git 에 올리지 않음)
spark_drivers/
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`JAVA_HOME is not set`|Java 미설치 또는 환경변수 없음|`brew install openjdk@17` + `.zshrc` 설정|
|`ClassNotFoundException: org.postgresql.Driver`|JAR 없음|`spark_drivers/` 다운로드 + 볼륨 마운트 확인|
|Spark 컨테이너 시작 직후 종료|JAR 폴더째 마운트로 엔진 파일 덮어씌워짐|파일 대 파일로 마운트|
|`--packages` 매번 느림|Maven 에서 반복 다운로드|JAR 미리 받아서 `--jars` 로 지정|