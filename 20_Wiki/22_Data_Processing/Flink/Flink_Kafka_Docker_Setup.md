---
aliases:
  - Kafka KRaft Setup
  - 카프카 도커 설정
tags:
  - Kafka
  - Docker
  - Flink
related:
  - "[[Flink_Docker_Setup(PyFlink)]]"
  - "[[Spark_Kafka_Docker_Setup]]"
  - "[[Apache_Kafka_Intro]]"
---
#  Kafka (KRaft Mode) & Flink Docker Setup

## 아키텍처 요약 (Architecture)

* **Kafka:** 최신 **KRaft 모드**를 사용 (Zookeeper 제거됨 ).
* **Flink:** **Session Mode**로 구성 (개발 편의성 최적화).
* **Network:** `INTERNAL`(19092)은 Flink용, `PLAINTEXT`(9092)는 로컬 터미널용.

---
## 핵심 설정 파일 (Configuration)

### ① `docker-compose.yml` (KRaft + Flink)

Zookeeper 없이 Kafka만 단독으로 실행되며, Flink 1.18.1과 연동됩니다.

```yaml
version: "3.8"

services:

  # =========================================
  # 1. Kafka (Apache 공식 이미지, KRaft 모드)
  # =========================================
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      # [필수] Kafka 노드 ID (KRaft 모드에서 고유해야 함)
      KAFKA_NODE_ID: 1

      # [필수] KRaft 역할 설정 (브로커 + 컨트롤러)
      KAFKA_PROCESS_ROLES: broker,controller

      # [필수] 리스너 정의
      # - PLAINTEXT   : 외부(localhost) 접속용
      # - INTERNAL    : Docker 내부 통신용
      # - CONTROLLER  : KRaft 컨트롤러 통신용
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093,INTERNAL://:19092

      # ⭐ 가장 중요 ⭐
      # 클라이언트가 접속할 실제 주소
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,INTERNAL://kafka:19092

      # 리스너 보안 프로토콜 매핑
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,INTERNAL:PLAINTEXT

      # 브로커 간 통신 리스너
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL

      # KRaft 쿼럼 설정 (노드ID@호스트:컨트롤러포트)
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093

      # [싱글 노드 필수 설정]
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1

    volumes:
      # Kafka 데이터 저장 경로
      # ⚠ Bitnami 이미지 경로(/bitnami/kafka) 아님
      - kafka_data:/var/lib/kafka/data

  # =========================================
  # 2. Flink JobManager
  # =========================================
  jobmanager:
    build: .                # Dockerfile 기반으로 Flink 이미지 빌드
    container_name: jobmanager
    command: jobmanager
    ports:
      - "8081:8081"         # Flink Web UI
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: jobmanager
    volumes:
      # 실습 코드 / 데이터 공유
      - ./playground:/opt/flink/playground

  # =========================================
  # 3. Flink TaskManager
  # =========================================
  taskmanager:
    build: .
    container_name: taskmanager
    command: taskmanager
    depends_on:
      - jobmanager
    environment:
      FLINK_PROPERTIES: |
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2
    volumes:
      - ./playground:/opt/flink/playground

volumes:
  kafka_data:
```

### ② `Dockerfile` (디버깅 도구 포함)


```Dockerfile
# 1. 베이스 이미지 (M1/M2 지원 버전)
FROM flink:1.18.1

USER root

# 2. 필수 패키지 + 빌드 도구 설치 (여기가 핵심!)
# build-essential: pemja 컴파일을 위한 gcc 등 설치
# openjdk-11-jdk-headless: 헤더 파일(jni.h)이 포함된 JDK 설치
RUN sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list && \
    sed -i 's/security.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-dev \
    build-essential \
    openjdk-11-jdk-headless \
    libssl-dev zlib1g-dev libffi-dev \
    ca-certificates \
    curl \
    netcat-openbsd \
    iputils-ping && \
    ln -s /usr/bin/python3 /usr/bin/python && \
    rm -rf /var/lib/apt/lists/*

# [중요] JAVA_HOME을 새로 설치한 JDK(arm64) 경로로 변경해줍니다.
# 그래야 pemja가 "아! 여기 헤더 파일 있구나!" 하고 찾습니다.
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-arm64

# 3. PyFlink 설치
RUN pip3 install --upgrade pip
RUN pip3 install --no-cache-dir --default-timeout=1000 \
    apache-flink==1.18.1 \
    typing_extensions

# 커넥터 다운로드
RUN curl -o /opt/flink/lib/flink-sql-connector-kafka-3.0.0-1.18.jar \
    https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.0.0-1.18/flink-sql-connector-kafka-3.0.0-1.18.jar

# 4. 환경 변수
ENV PYTHON_EXECUTABLE=/usr/bin/python3
ENV PYFLINK_CLIENT_EXECUTABLE=/usr/bin/python3

USER flink
WORKDIR /opt/flink
```

---
## Application Mode로 전환하는 법 (Production)

지금은 개발 편의를 위해 **Session Mode**를 쓰고 있습니다. 위에 Docker-compose.yml는 Session Mode
만약 **상용 배포(Production)** 를 위해 **Application Mode**로 바꾸고 싶다면 아래 2가지를 수정하세요.

### Step 1. Dockerfile 수정 (코드 내장)

코드를 컨테이너 안에 **미리 복사(COPY)** 해둬야 합니다.

```Dockerfile
# ... (기존 설정 동일) ...

# [추가] 내 코드를 이미지 안으로 복사
COPY src/kafka_word_count.py /opt/flink/usrlib/my_job.py
```

### Step 2. docker-compose.yml 수정 (명령어 변경)

`jobmanager`가 켜지자마자 **즉시 잡을 실행**하도록 `command`를 바꿉니다.

```yaml
jobmanager:
    build: .
    # [변경] standalone-job으로 실행하며, 파이썬 파일 경로를 지정
    command: standalone-job --python /opt/flink/usrlib/my_job.py
    ports:
      - "8081:8081"
    # ... (나머지 동일)
```

### 💡 차이점 요약

|**모드**|**코드 위치**|**실행 시점**|**용도**|
|---|---|---|---|
|**Session** (현재)|로컬 볼륨 (`./playground`)|컨테이너 켜고 나서 `docker exec ...`|**개발 / 테스트**|
|**Application**|이미지 내부 (`/opt/...`)|컨테이너 켜지자마자 **자동 실행**|**배포 / 운영**|


---
## 코드로 확인해보기 -> 실행 및 확인 방법 (터미널 3개 필요)

1. .py 파일 만들기(kafka_word_count_job.py)

### Terminal 1: 결과 확인용 (Consumer)

결과가 나오는 `output-topic`을 실시간으로 감시합니다.

```bash
# Kafka 컨테이너 접속
docker exec -it [Kafka컨테이너ID] bash

# 결과 토픽 구독 (데이터가 들어오면 바로 출력됨)
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic output-topic
```

### Terminal 2: Flink Job 실행

JobManager에 들어가서 위에서 작성한 파이썬 코드를 실행합니다.

```bash
# JobManager 컨테이너 접속
docker exec -it [JobManager컨테이너ID] bash

# 코드 실행
python /opt/flink/playground/src/kafka_word_count_job.py
```

### 3️⃣ Terminal 3: 데이터 입력용 (Producer)

Kafka 컨테이너에 하나 더 접속해서 `input-topic`에 말을 겁니다.

```bash
# Kafka 컨테이너 접속
docker exec -it [Kafka컨테이너ID] bash

# 입력 토픽에 데이터 전송
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic input-topic
```

### **Terminal 3**에 입력하면 -> **Terminal 1**에 결과가 나와야 합니다.

**입력 (Terminal 3):**

```text
> hello flink
> kafka is fun
> hello kafka
```

**출력 (Terminal 1):**

```text
hello:1
flink:1
kafka:1
is:1
fun:1
hello:2  <-- 누적 계산됨!
kafka:2  <-- 누적 계산됨!
```

---
## [Troubleshooting] M1/M2 맥북 빌드 실패 원인 분석

**핵심 요약:** PyFlink가 사용하는 **`pemja`** 라이브러리가 ARM(Apple Silicon)용 설치 파일(Wheel)이 없어서 **직접 컴파일**을 시도했으나, **필수 재료(헤더 파일, 컴파일러)** 가 없어서 실패했음.

#### 문제의 발단 (`pemja`와 ARM 아키텍처)

- **상황:** PyFlink는 파이썬과 자바를 연결하기 위해 `pemja`라는 다리를 사용함.
- **원인:** 윈도우나 인텔 맥북(amd64)은 누가 미리 만들어둔 설치 파일(Wheel)이 있어서 그냥 가져다 쓰면 됨. 하지만 **M1/M2(arm64)용 파일은 없어서, 설치 도중 즉석에서 소스 코드를 빌드(컴파일)해야 했음.**

#### 왜 에러가 났나? (Missing Dependencies)

컴파일을 하려면 **"도구(GCC)"** 와 **"설계도(Header File)"** 가 필요한데, 가벼운 기본 Flink 이미지에는 이게 다 빠져 있었음.

- **`gcc` 없음:** C/C++ 코드를 번역할 도구가 없음 (`build-essential` 누락).
- **`jni.h` 없음:** 자바와 연결할 때 필요한 핵심 설계도 파일이 없음. 기본 이미지는 실행만 가능한 **JRE(Runtime)** 버전이라, 개발용 도구인 **JDK(Development Kit)**에만 들어있는 헤더 파일이 없었음.
- **에러 메시지:** `Include folder ... doesn't exist` (설계도 어딨어? 못 찾겠는데?)

#### 해결책 (Solution)

`Dockerfile`에 아래 3가지 조치를 취해서 **"직접 건설 가능한 환경"** 을 만들어줌.

1. **`build-essential` 설치:** 컴파일러(건설 장비) 반입.
2. **`openjdk-11-jdk-headless` 설치:** 헤더 파일(설계도) 반입.
3. **`ENV JAVA_HOME=...` 설정:** 컴파일러한테 "설계도 저기 있으니까 갖다 써"라고 위치 알려줌.
