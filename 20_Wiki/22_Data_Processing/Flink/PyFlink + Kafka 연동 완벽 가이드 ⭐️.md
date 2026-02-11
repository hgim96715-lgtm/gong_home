---
aliases:
  - PyFlink Kafka 연동
  - Flink Kafka Troubleshooting
  - PyFlink 실전 매뉴얼
tags:
  - Kafka
  - PyFlink
related:
  - "[[PyFlink_Kafka_Source]]"
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink + Kafka 연동 완벽 가이드 ⭐️]]"
  - "[[PyFlink_Kafka_코드해부_Common ⭐️]]"
  - "[[PyFlink_Kafka_Sink_Guide]]"
---
# 🔥 PyFlink + Kafka 연동 완벽 가이드 (Integrated)

> [!NOTE] 문서의 목적
> 이 문서는 Flink와 Kafka를 연동하면서 겪을 수 있는 **모든 환경적 문제(JAR, Network, Docker)**와 **두 가지 실행 패턴(Source vs Sink)**을 완벽하게 정리하여, 나중에 다시 볼 때 **단 한 번의 실패 없이 실행**하기 위함이다.

---
## Architecture Overview (전체 흐름)

**"데이터의 흐름: Kafka (Producer) -> Flink (Consumer) -> Processing -> Output"**

1.  **Kafka Container:** 메시지를 받아주는 큐 (Broker)이자 공구함.
2.  **JobManager:** 파이썬 코드를 받아서 작업을 배분하는 리더.
3.  **TaskManager:** 실제로 데이터를 처리하고 로그를 찍거나 다시 Kafka로 보내는 일꾼.

---
##  필수 준비물 (Prerequisites) & 트러블슈팅

### ① Connector JAR 파일 (가장 중요 ⭐️)

PyFlink는 깡통이라 Kafka와 대화하려면 **통역사(JAR)** 가 반드시 필요합니다.

* **파일명:** `flink-sql-connector-kafka-3.1.0-1.18.jar`
* **위치:** `/opt/flink/lib/` (컨테이너 내부)
* **필수 확인:**
    * `3.0.0` 버전 없음 (404 Error). 반드시 **`3.1.0`** 이상.
    * **파일 크기 확인 필수!** (554 Bytes = 가짜 HTML 파일 / **5.3 MB 이상 = 정상**)

#### 🚨 Trouble: "어? 아까 있던 파일이 사라졌어요!" (초기화 문제)

> **상황:** 방금 전까지 잘 실행됐는데, `docker-compose down` 후 다시 켰더니 JAR 파일이 없다는 에러(`JAR file does not exist`)가 뜸.

* **원인:** Docker 컨테이너는 **"휘발성(Ephemeral)"** 입니다. 재시작하면 이미지 상태로 초기화되어 수동으로 받은 파일은 증발합니다.

#### ✅ Solution 1. 지금 당장 다운로드 (1회성 응급처치)

컨테이너 실행 중일 때 다시 채워 넣는 방법입니다.

```bash
# 1. lib 폴더로 이동
cd /opt/flink/lib

# 2. JAR 파일 다시 다운로드 (curl -L 옵션 권장)
curl -O [https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.1.0-1.18/flink-sql-connector-kafka-3.1.0-1.18.jar](https://repo.maven.apache.org/maven2/org/apache/flink/flink-sql-connector-kafka/3.1.0-1.18/flink-sql-connector-kafka-3.1.0-1.18.jar)

# 3. 잘 받아졌는지 크기 확인 (5MB 이상이어야 함!)
ls -lh flink-sql-connector-kafka-3.1.0-1.18.jar
```

#### ✅ Solution 2. 영구적으로 해결하기 (권장: Volume 마운트)

내 컴퓨터에 파일을 받아두고 컨테이너에 연결하면 재시작해도 사라지지 않습니다.

```yaml
# docker-compose.yml
volumes:
  - ./playground/src:/opt/flink/playground/src
  # 👇 이 줄 추가! (내 컴퓨터의 lib 폴더를 컨테이너 안으로 투영)
  - ./lib/flink-sql-connector-kafka-3.1.0-1.18.jar:/opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

### ② Docker Network 설정 (Advertised Listeners)

Kafka가 "내 주소는 여기야"라고 알려주는 명함 설정입니다.

```yaml
environment:
  # 도커 내부(Flink용): 'kafka:9092'
  # 내 컴퓨터(Host용): 'localhost:29092'
  - KAFKA_ADVERTISED_LISTENERS=INTERNAL://kafka:9092,EXTERNAL://localhost:29092
```

---
## Step 1: Kafka 토픽 생성 & 도구 이해

Kafka 컨테이너에 들어가서 데이터를 담을 그릇(Topic)을 만듭니다.

```bash
# 내 컴퓨터 터미널에서 실행
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-topics.sh \
  --create \
  --bootstrap-server localhost:9092 \
  --topic input-topic
```

###  Q1. 왜 여기선 `localhost:9092`이고, Python에선 `kafka:9092` 인가요?

명령어를 **어디서 실행하느냐(Context)** 에 따라 부르는 주소가 다릅니다.

|**상황**|**명령어 실행 위치**|**접속 주소**|**의미**|**결과**|
|---|---|---|---|---|
|**토픽 생성**|**Kafka 컨테이너** (본인)|`localhost:9092`|"나 자신(내 집)한테 말 걸기"|**O (정상)**|
|**Flink 코드**|**Flink 컨테이너** (타인)|`localhost:9092`|"내 뱃속(Flink)에서 Kafka 찾기"|**X (에러)**|
|**Flink 코드**|**Flink 컨테이너** (타인)|`kafka:9092`|"저기 옆집(Kafka) 부르기"|**O (정상)**|

### Q2. `kafka-topics.sh` 이건 어디서 튀어나온 거죠?

이미지(`apache/kafka`)를 받는 순간 `/opt/kafka/bin/` 폴더에 미리 설치되어 오는 **공구 상자(Tool Kit)** 입니다.

```bash
# 눈으로 확인하기
docker exec -it apache-flink-kafka-1 ls -F /opt/kafka/bin/
```

- `/opt/`: 추가 프로그램 설치 구역
- `kafka/`: Kafka 설치 폴더
- `bin/`: 실행 파일(Binary) 모음


## **토픽 목록 확인 (List)**

- 현재 생성된 모든 우체통(토픽) 이름을 확인합니다.

```bash
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-topics.sh \
--list \
--bootstrap-server kafka:9092
```

---
## Execution Patterns (실행 패턴 비교)  ⭐️⭐️

Flink Job이 데이터를 **어디로 뱉느냐**에 따라 **확인하는 화면과 터미널 구성**이 완전히 다릅니다.

|**구분**|**Pattern A: Basic Debugging**|**Pattern B: Real-World Pipeline (Sink)**|
|---|---|---|
|**코드 특징**|`stream.print()` 사용|`stream.sink_to(kafka_sink)` 사용|
|**목적**|개발 중 데이터가 잘 읽히나 확인|데이터를 가공해서 다시 Kafka로 저장|
|**확인 위치**|**TaskManager 로그** (Docker Log)|**Kafka Output Topic** (Consumer)|
|**필요 파일**|`kafka_source.py`|`kafka_copy_job.py` (Process & Sink)|

---
## Pattern A: Basic Debugging (단순 로그 확인)

- **상황:** Kafka에서 데이터를 읽어서(`Source`) 화면에 뿌려보기(`Print`).
- **준비물:** 터미널 3개

### [창 1] 로그 감시 (TaskManager)

출력(`print()`)은 내 화면이 아니라 **일하는 놈(TaskManager)** 화면에 뜹니다.

```bash
docker logs -f apache-flink-taskmanager-1
```

### [창 2] 데이터 전송 (Producer)

```bash
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic input-topic
```

### [창 3] Job 실행 (Submit)

JobManager에 들어가서 실행합니다.

```bash
# JobManager 접속
docker exec -it apache-flink-jobmanager-1 bash

# Job 제출 (표준 명령어)
/opt/flink/bin/flink run \
  -py /opt/flink/playground/src/kafka_trigger.py \
  -j /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

---
## Pattern B: Real-World Pipeline (Sink to Kafka)

- **상황:** Kafka(`input`) -> Flink 가공(대문자 변환) -> Kafka(`output`) 저장.
- **핵심:** 로그가 아니라 **최종 목적지(Output Topic)** 를 훔쳐봐야 합니다.
- **준비물:** 터미널 3개

### [창 1] 결과 감시 (Consumer)

가장 먼저 켜두세요. 결과가 도착하면 바로 뜨도록 대기합니다.

(로그 보는 명령어가 아니라, **Kafka 토픽을 훔쳐보는 명령어**를 써야 합니다.)

```bash
# output-topic을 계속 지켜보다가 데이터가 오면 출력해라!
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic output-topic
```

### [창 2] 데이터 전송 (Producer) 

여기에 입력하면 Flink가 가공해서 [창 1]로 보낼 겁니다.

```bash
# input-topic에 데이터를 쏘는 역할
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic input-topic
```

### [창 3] Job 실행 (Submit) 

JobManager에 명령을 내립니다. (파일명이 **`만든파일.py`** 인지 확인!)

```bash
# 1. JobManager 접속
docker exec -it apache-flink-jobmanager-1 bash

# 2. 실행 (JAR 파일 필수!)
/opt/flink/bin/flink run \
  -py /opt/flink/playground/src/kafka_window_function.py \
  -j /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

---
### Action Scenario (성공 시나리오)

1. **[창 3]** 에서 실행 명령을 내린다. (`Job submitted...` 메시지 확인)
2. **[창 2]** 에 `hello world` 라고 입력하고 **엔터**.
3. **[창 1]** 에 `Processed: Hello World` ( 로직 적용됨)가 짠! 하고 나타나면 성공! 🎉

---
##  Job 관리 및 종료 (Docker 환경) 

테스트가 끝났거나 코드를 수정해서 다시 올리려면, **반드시 기존 Job을 종료**해야 합니다.
Docker를 쓰고 계시므로, 명령어 앞에 `docker exec`를 붙여서 컨테이너에게 명령을 전달해야 합니다.

### 1단계: 실행 중인 Job ID 찾기 🔍

컨테이너 안에 직접 들어가지 않고, 밖에서 명령만 툭 던져서 확인하는 방법입니다.

```bash
# 문법: docker exec <컨테이너명> <명령어>
docker exec -it apache-flink-jobmanager-1 ./bin/flink list
```

**출력 예시:** `Waiting for response...` `------------------ Running/Restarting Jobs -------------------` `10.10.10.10 : e7a96d6ab3120f3ed44db75aedd4c263 : PyFlink Window Example (RUNNING)`

👉 중간에 있는 긴 문자열(`e7a...`)이 **Job ID**입니다.

### 2단계: Job 강제 종료 (Cancel) 

복사한 Job ID를 넣어서 취소 명령을 날립니다.

```bash
# 문법: docker exec -it <컨테이너명> ./bin/flink cancel <JOB_ID>
docker exec -it apache-flink-jobmanager-1 ./bin/flink cancel e7a96d6ab3120f3ed44db75aedd4c263
```

**성공 시 출력:** `Cancelling job e7a96d6ab3120f3ed44db75aedd4c263.` `Job has been cancelled.`


### 💡 꿀팁: Web UI가 제일 편해요

명령어가 너무 길고 복잡하죠? 개발 중에는 그냥 **Web Dashboard (http://localhost:8081)** 접속 -> 해당 Job 클릭 -> 우측 상단 **`Cancel Job`** 버튼


