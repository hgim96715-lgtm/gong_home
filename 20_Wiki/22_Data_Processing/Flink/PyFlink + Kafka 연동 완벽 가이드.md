---
aliases:
  - PyFlink Kafka 연동
  - Flink Kafka Troubleshooting
  - PyFlink 실습
tags:
  - Kafka
  - Flink
related:
  - "[[Flink_Kafka_Source]]"
  - "[[00_Apache Flink_HomePage]]"
---
#  PyFlink + Kafka 연동 & 트러블슈팅 로그

## Architecture Overview

**"데이터의 흐름: Kafka (Producer) -> Flink (Consumer) -> TaskManager Log (Output)"**

1.  **Kafka Container:** 메시지를 받아주는 큐 (Broker).
2.  **JobManager:** 파이썬 코드를 받아서 작업을 배분하는 리더.
3.  **TaskManager:** 실제로 데이터를 처리하고 로그를 찍는 일꾼.

----
##  필수 준비물 (Prerequisites)

### ① Connector JAR 파일 (필수!)

PyFlink는 깡통이라 Kafka와 대화하려면 **통역사(JAR)** 가 필요함.
* **파일명:** `flink-sql-connector-kafka-3.1.0-1.18.jar`
* **위치:** `/opt/flink/lib/` (컨테이너 내부)
* **주의사항:**
    * `3.0.0` 버전은 존재하지 않음 (404 Error). 반드시 **`3.1.0`** 이상 사용.
    * **가짜 파일 주의:** Dockerfile에 `curl` 명령어를 넣어뒀더라도, URL이 틀리거나 리다이렉트 문제로 **554 Bytes 짜리 에러 메시지(HTML)** 만 받아지는 경우가 있음. (반드시 확인 필요!)

#### 💡 실행 시 `Java dependencies...` 에러가 난다면?

만약 파이썬 실행 시 아래와 같은 에러가 발생한다면, JAR 파일이 로딩되지 않은 것임.

`{text}Error: "The Java dependencies could be specified via command line argument '--jarfile' or the config option 'pipeline.jars'"`

👉 해결책: `flink run` 명령어로 JAR를 강제 주입해서 실행

```bash
/opt/flink/bin/flink run \
  -py /opt/flink/playground/src/kafka_source.py \
  -j /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

- **`-py`**: 실행할 파이썬 파일의 경로.
- **`-j`**: (Jar) 함께 데려갈 JAR 파일 경로. **(여기에 명시하면 무조건 로딩됩니다!)**

#### (최종 확인)

**가짜 파일**이 또 받아진 건 아닌지 의심된다면 크기를 확인

```bash
ls -lh /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

- **554 Bytes (KB 단위):** ❌ 실패! (HTML 에러 페이지임. 다시 받아야 함)
- **5.3 MB 근처:** ✅ 정상! (위 `flink run` 명령어로 실행하면 100% 됨)

### ② Docker Compose 설정 (네트워크)

Kafka가 "내 주소는 여기야"라고 알려주는 **Advertised Listeners** 설정이 핵심.

```yaml
# docker-compose.yml (Kafka 부분)
environment:
  # 도커 내부(Flink)는 'kafka:9092', 내 컴퓨터(Host)는 'localhost:29092'로 접속
  - KAFKA_ADVERTISED_LISTENERS=INTERNAL://kafka:9092,EXTERNAL://localhost:29092
  - KAFKA_LISTENERS=INTERNAL://0.0.0.0:9092,EXTERNAL://0.0.0.0:29092
```

---
## 실행 가이드 (Step-by-Step)

### Step 1. Kafka 토픽 생성

Kafka 컨테이너에 들어가서 데이터를 담을 그릇(Topic)을 만든다.

```bash
# 내 컴퓨터 터미널
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-topics.sh \
  --create \
  --bootstrap-server localhost:9092 \
  --topic input-topic
```

### Step 2. Flink Job 실행 (Consumer)

**중요:** `python3` 명령어가 아니라 **`flink run`** 명령어를 써야 JAR 로딩 문제가 없음.


```bash
# JobManager 컨테이너 접속
docker exec -it apache-flink-jobmanager-1 bash

# Job 제출 (JAR 파일 경로 명시)
/opt/flink/bin/flink run \
  -py /opt/flink/playground/src/kafka_source.py \
  -j /opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar
```

- 결과: `Job has been submitted with JobID ...` 뜨면 성공.

### Step 3. 결과 로그 확인 (모니터링)

출력(`print()`)은 내 화면이 아니라 **일하는 놈(TaskManager)** 화면에 뜸.
새로운 터미널 창 열어서 확인 

```bash
# 새 터미널 창 열기
docker logs -f apache-flink-taskmanager-1
```

### Step 4. 데이터 전송 테스트 (Producer)

새로운 터미널 창을열어서 입력하고, `>` 가 생기면 원하는 것 입력 
Kafka에 메시지를 넣으면, Step 3의 화면에 실시간으로 떠야 함.

```bash
# 또 다른 새 터미널 창 열기
docker exec -it apache-flink-kafka-1 /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic input-topic
```







