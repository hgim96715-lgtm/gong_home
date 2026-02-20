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
  - "[[Kafka_Python_Producer_Basic]]"
  - "[[Kafka_Python_Consumer_Basic]]"
  - "[[Spark_JDBC_Postgres_Guide]]"
---

## 개념 한 줄 요약 (Concept Summary)

**"코드를 실행하는 위치(Where am I?)가 '내 컴퓨터(Host)'인지 '컨테이너 안(Container)'인지에 따라 접속 주소와 포트가 달라진다."**
마치 아파트 단지(Docker) 밖에서 친구를 부를 때와, 단지 안에서 이웃을 부를 때 주소가 다른 것과 같다.

---

## 왜 필요한가? (Why)

**문제점:**
- 로컬 터미널에서 파이썬 코드를 돌릴 때는 `localhost:29092`로 잘 되는데, 막상 같은 코드를 도커 컨테이너(Airflow, Spark 등)에 올리면 `Connection Refused` 에러가 난다.
- 반대로, 도커 내부 설정(`kafka:9092`)을 그대로 내 컴퓨터에서 쓰면 접속이 안 된다.

**해결책:**
- 도커 컨테이너는 **외부용 문(Port)** 과 **내부용 문(Port)** 을 따로 가질 수 있음을 이해해야 한다.
- 내가 **어디 서 있는지**에 따라 올바른 문(주소)을 골라야 통신이 뚫린다.

---

## 실전 맥락 (Practical Context)

**데이터 엔지니어링 상황별 접속 공식:**

1.  **내 맥북(Host)에서 실행할 때:** (예: Streamlit 대시보드 실행, `fake_log_producer.py`, DBeaver/DataGrip 등 DB 툴 접속)
    - 우리는 도커라는 아파트 단지 **"외부인"** 이다.
    - 무조건 **`localhost`** 와 **`외부 포트(External Port)`** 를 써야 한다.
    - 예: `localhost:29092` (Kafka), `localhost:5433` (PostgreSQL), `localhost:8080` (Airflow Web)

2.  **도커 컨테이너 안에서 실행할 때:** (예: Flink Job, Spark Worker, Airflow DAG 내부 로직)
    - 우리는 이미 아파트 단지 **"입주민"** 이다.
    - **`서비스 이름(Service Name)`** 과 **`내부 포트(Internal Port)`** 를 써야 한다.
    - 예: `kafka:9092` (Kafka), `spark-master:7077` (Spark), `postgres:5432` (PostgreSQL)

---

## Code Core Points (핵심 설정 분석)

도커 컴포즈에서 이 "두 개의 문"이 어떻게 만들어지는지 보자.

```yaml
# docker-compose.yml (Kafka 예시)
services:
  kafka:
    image: bitnami/kafka:latest
    ports:
      - "29092:9092"  # ⭐️ 핵심: "외부포트:내부포트" 매핑
    environment:
      # 리스너(문) 2개 선언: INTERNAL(내부용), EXTERNAL(외부용)
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,EXTERNAL://:29092
      
      # 주소표(광고): 
      # "내부 애들은 kafka:9092로 오고, 외부(맥북) 애들은 localhost:29092로 와라"
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:29092
```

### 상세 분석: 아파트 비유 (Detailed Analysis)

|**구분**|**내 맥북 (Host) 💻**|**도커 컨테이너 (Container) 🐳**|
|---|---|---|
|**나의 신분**|외부 손님|아파트 입주민|
|**접근 방식**|아파트 정문(`localhost`)으로 가서 동호수 호출|단지 내 방송으로 "101동(`kafka`)으로 와"|
|**사용 주소**|**`localhost`**|**`kafka` (컨테이너명)**|
|**사용 포트**|**`29092`** (외부에 노출된 문)|**`9092`** (원래 자기 문)|
|**파이썬 코드**|`bootstrap_servers=['localhost:29092']`|`bootstrap_servers=['kafka:9092']`|