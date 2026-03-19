---
aliases:
  - Docker Network
  - localhost vs container_name
  - 도커 외부접속 내부통신
  - 포트포워딩
tags:
  - Docker
  - Kafka
related:
  - "[[00_Kafka_HomePage]]"
  - "[[Kafka_Python_Producer]]"
  - "[[Kafka_Python_Consumer]]"
  - "[[Spark_JDBC_PostgreSQL]]"
  - "[[Docker_Compose_Setup]]"
  - "[[Apache_Kafka_Concept]]"
  - "[[Linux_Network_Basic]]"
  - "[[Linux_Port]]"
---
# Docker_Host_vs_Internal_Network

## 한 줄 요약

```
코드를 실행하는 위치가 "내 컴퓨터(Host)" 인지 "컨테이너 안" 인지에 따라
접속 주소와 포트가 달라진다

아파트 단지(Docker) 밖에서 친구를 부를 때와
단지 안에서 이웃을 부를 때 주소가 다른 것과 같다
```

---

---

# ① 왜 헷갈리는가?

```
문제 1:
  로컬 터미널에서 Python 코드 실행 → localhost:9093 으로 잘 됨
  같은 코드를 Airflow / Spark 컨테이너에 올리면 → Connection Refused

문제 2:
  컨테이너 내부 설정(kafka:9092) 을 그대로 맥북에서 쓰면 → 접속 안 됨

원인:
  컨테이너는 외부용 문(포트) 과 내부용 문(포트) 이 따로 있음
  내가 어디 서 있는지에 따라 올바른 주소를 골라야 함
```

---

---

# ② 상황별 접속 공식

## 맥북(Host) 에서 실행할 때

```
신분: 아파트 단지 외부인
→ localhost + 외부 포트(External Port) 사용

예:
  Kafka       localhost:9093   (hospital 프로젝트)
  PostgreSQL  localhost:5434
  Airflow UI  localhost:8084
  Spark UI    localhost:8083
  Superset    localhost:8089
```

```python
# 맥북 터미널에서 실행하는 Python 코드
producer = KafkaProducer(bootstrap_servers=["localhost:9093"])
```

## 컨테이너 안에서 실행할 때

```
신분: 아파트 단지 입주민
→ 서비스 이름(컨테이너명) + 내부 포트 사용

예:
  Kafka       kafka:9092
  PostgreSQL  postgres:5432
  Spark       spark-master:7077
```

```python
# Airflow DAG / Spark Job 코드 (컨테이너 안에서 실행)
producer = KafkaProducer(bootstrap_servers=["kafka:9092"])
```

## 한눈에 비교

|구분|맥북 (Host)|컨테이너 (Container)|
|---|---|---|
|신분|외부 손님|아파트 입주민|
|Kafka 주소|`localhost:9093`|`kafka:9092`|
|PostgreSQL|`localhost:5434`|`postgres:5432`|
|사용 주소|`localhost`|서비스명 (컨테이너명)|
|사용 포트|외부 포트 (포트포워딩)|내부 포트 (원래 포트)|

---

---

# ③ docker-compose.yml 에서 두 개의 문

```yaml
services:
  kafka:
    image: apache/kafka:3.7.0
    ports:
      - "9093:9092"    # "외부포트:내부포트"
                       # 맥북 9093 → 컨테이너 9092
```

```
포트포워딩 구조:

  맥북 9093 ──→ [Docker 브릿지] ──→ 컨테이너 9092
                                         ↑
                               Kafka 실제 실행 포트
```

---

---

# ④ KAFKA_LISTENERS vs KAFKA_ADVERTISED_LISTENERS ⭐️

```
KAFKA_LISTENERS          Kafka 가 실제로 열고 있는 소켓 주소
                         "나 이 주소에서 연결 기다리고 있을게"

KAFKA_ADVERTISED_LISTENERS  클라이언트에게 알려주는 접속 주소
                         "나한테 연결할 때 이 주소로 와"
```

## Apache 공식 이미지 설정 (우리 프로젝트)

```yaml
# docker-compose.yml
environment:
  KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
  KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
```

```
KAFKA_LISTENERS:
  PLAINTEXT://:9092   → 모든 인터페이스의 9092 포트에서 연결 대기
  CONTROLLER://:9093  → 컨트롤러 통신용 9093 포트

KAFKA_ADVERTISED_LISTENERS:
  PLAINTEXT://kafka:9092
  → 클라이언트에게 "kafka:9092 로 연결해" 라고 알려줌
  → Docker 네트워크 안에서 kafka 는 서비스명으로 DNS 해석됨
  → 컨테이너끼리는 kafka:9092 로 통신 가능

⚠️ localhost 로 설정하면 안 되는 이유:
  PLAINTEXT://localhost:9092 로 설정하면
  다른 컨테이너(Airflow, Spark)에서 연결 시
  localhost = 자기 자신 → Kafka 컨테이너가 아님 → 연결 실패
```

## ⚠️ KAFKA_ADVERTISED_LISTENERS 는 절대 바꾸면 안 됨

```
자주 하는 실수:
  맥북에서 kafka:9092 로 접속하고 싶어서
  ADVERTISED_LISTENERS=localhost:9092 로 변경 시도

왜 안 되는가:
  Spark / Airflow 컨테이너가 Kafka 에 bootstrap 연결 후
  "localhost:9092 로 메시지 보내" 라고 안내받음
  컨테이너 안에서 localhost = 자기 자신 (Kafka 아님)
  → 연결 실패 → 컨테이너끼리 통신 전부 깨짐

올바른 해결:
  ADVERTISED_LISTENERS=kafka:9092 그대로 유지
  docker-compose.yml 에 9092 포트 추가로 맥북 접근 허용
```


```yaml
# 맥북에서 kafka:9092 접속하고 싶으면
# ADVERTISED_LISTENERS 변경 ❌
# 포트 추가 ✅

kafka:
  ports:
    - "9093:9092"   # 기존
    - "9092:9092"   # 추가 → 맥북에서 9092 직접 접근 가능
  environment:
    KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092  # 그대로 유지
```

```
포트 추가 후:
  맥북    kafka:9092 → 127.0.0.1:9092 → 포트포워딩 → 컨테이너 9092 ✅
  컨테이너 kafka:9092 → Docker DNS → 컨테이너 9092 ✅
  → 맥북 / 컨테이너 모두 kafka:9092 로 통일 가능

실제로 발행완료 로그가 떴는데 CLI Consumer 에서 메시지가 없던 이유:
  Producer → kafka:9093 (bootstrap 연결 성공)
  브로커: "kafka:9092 로 메시지 보내"
  Producer → kafka:9092 = 127.0.0.1:9092 → 9092 포트 닫혀있음
  → 실제 메시지 전송 실패
  → KafkaProducer 는 에러 없이 넘어감 → 발행완료 로그가 거짓말
```

## Bitnami 이미지 설정 방식 (참고)

```yaml
# Bitnami 방식 (유료 전환으로 미사용)
environment:
  - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,EXTERNAL://:29092
  - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:29092
```

```
Bitnami 는 리스너를 2개 선언:
  PLAINTEXT (내부용): kafka:9092
  EXTERNAL  (외부용): localhost:29092

Apache 공식은 리스너 1개로 단순하게:
  PLAINTEXT://kafka:9092 하나로 통일
  맥북 → 컨테이너는 포트포워딩(9093:9092) 으로 해결
```

---

---

# ⑤ /etc/hosts 설정 — 맥북에서 kafka 이름 인식

```
문제:
  맥북에서 kafka:9092 쓰면
  → 맥북은 "kafka" 호스트명 모름 → NoBrokersAvailable 에러

해결:
  맥북 /etc/hosts 에 kafka = 127.0.0.1 등록
```

```bash
sudo vi /etc/hosts
# 아래 줄 추가
127.0.0.1 kafka
```

```
⚠️ /etc/hosts 등록 후에도 포트는 외부 포트 써야 함

  kafka:9092 → 127.0.0.1:9092 → ❌ (9092 는 컨테이너 내부 포트)
  kafka:9093 → 127.0.0.1:9093 → ✅ (9093 이 외부 노출 포트)

  docker-compose.yml: "9093:9092"
  9093 = 맥북 접속용 (외부)
  9092 = 컨테이너 내부 Kafka 포트

맥북에서:
  kafka:9093      ← /etc/hosts + 외부 포트
  localhost:9093  ← 직접 포트 (더 간단)

컨테이너 안에서:
  kafka:9092      ← Docker DNS + 내부 포트
```

>"9093:9092 니까 kafka:9092 쓰면 되겠지?" → ❌ 이런 생각하지말자!!

---

---

# ⑥ 실전 정리 — 프로젝트별 접속 주소

## Hospital 프로젝트

|서비스|맥북에서|컨테이너에서|
|---|---|---|
|Kafka|`localhost:9093`|`kafka:9092`|
|PostgreSQL|`localhost:5434`|`postgres:5432`|
|Spark|`localhost:8083`|`spark-master:7077`|
|Airflow|`localhost:8084`|-|
|Superset|`localhost:8089`|-|

## 서울역 프로젝트

|서비스|맥북에서|컨테이너에서|
|---|---|---|
|Kafka|`localhost:9092`|`kafka:9092`|
|PostgreSQL|`localhost:5433`|`postgres:5432`|
|Spark|`localhost:8080`|`spark-master:7077`|

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|맥북에서 `NoBrokersAvailable`|kafka 이름 인식 못 함|`/etc/hosts` 에 `127.0.0.1 kafka` 추가|
|컨테이너에서 `Connection Refused`|localhost 로 연결 시도|서비스명 (kafka:9092) 으로 변경|
|`ADVERTISED_LISTENERS localhost` 설정 시 에러|다른 컨테이너에서 localhost = 자기 자신|`PLAINTEXT://kafka:9092` 로 변경|
|포트 충돌|두 프로젝트가 같은 포트|hospital 9093, train 9092 로 분리|