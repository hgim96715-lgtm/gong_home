---
aliases:
  - 도커 컴포즈
  - docker compose
  - 컨테이너 오케스트레이션
  - yml 문법
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Volumes]]"
  - "[[Docker_Networking]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
---

# 🐳 Docker_Compose_Setup

## 개념 한 줄 요약

> **"컨테이너 3개를 따로따로 `docker run` 하지 말고, 파일 하나로 한 번에 켜고 끄자."** 
> `docker-compose.yml` 은 여러 컨테이너의 설정을 한 파일에 모아서 관리하는 설계도다.

```
docker run 3번  →  번거롭고 실수 많음
docker compose up  →  파일 하나로 전부 실행
```

---

---

# ① 전체 구조

```yaml
services:               # 실행할 컨테이너 목록
  서비스명1:
    image: ...
    ...
  서비스명2:
    image: ...
    ...

volumes:                # 영구 저장소 선언
  볼륨명1:
  볼륨명2:

networks:               # 네트워크 선언
  네트워크명:
```

---

---

# ② services — 컨테이너 설정

> **각 컨테이너를 어떻게 실행할지 정의하는 핵심 섹션.**

```yaml
services:
  postgres:                          # 서비스 이름 (컨테이너 간 DNS 이름)
    image: postgres:16               # 사용할 이미지
    container_name: train-postgres   # 실제 컨테이너 이름
    restart: always                  # 재시작 정책
    ports:
      - "5432:5432"                  # 호스트:컨테이너 포트포워딩
    environment:                     # 환경변수
      - POSTGRES_USER=${POSTGRES_USER}
    volumes:                         # 볼륨 연결
      - postgres_data:/var/lib/postgresql/data
    networks:                        # 네트워크 연결
      - train-network
    depends_on:                      # 의존 관계
      kafka:
        condition: service_healthy
    healthcheck:                     # 상태 체크
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

---

---

# ③ environment — 환경변수 주입

> **컨테이너 안에서 사용할 환경변수를 설정한다.** 민감한 값은 `.env` 파일에 분리해서 관리.

```yaml
# 방식 1: 직접 값 입력
environment:
  POSTGRES_USER: train_user
  POSTGRES_PASSWORD: train1234

# 방식 2: 리스트 형태
environment:
  - POSTGRES_USER=train_user
  - POSTGRES_PASSWORD=train1234

# 방식 3: .env 파일에서 가져오기 (권장) ⭐️
environment:
  - POSTGRES_USER=${POSTGRES_USER}
  - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
```

```
# .env 파일
POSTGRES_USER=train_user
POSTGRES_PASSWORD=train1234

→ docker-compose.yml 과 같은 폴더에 있으면 자동으로 읽음
→ .gitignore 에 추가해서 GitHub 에 올라가지 않도록
```

---

---

# ④ depends_on — 실행 순서 제어

> **다른 컨테이너가 준비된 후에 실행되도록 순서를 지정한다.**

```yaml
services:
  kafka:
    image: apache/kafka:3.7.0
    healthcheck:
      test: ["CMD", "/opt/kafka/bin/kafka-topics.sh", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 5s
      retries: 5

  producer:
    image: my-producer
    depends_on:
      kafka:                          # kafka 가 준비된 후에 실행
        condition: service_healthy    # healthcheck 통과 후
```

## depends_on condition 종류

|condition|의미|
|---|---|
|`service_started`|컨테이너가 시작되기만 하면 됨 (기본값)|
|`service_healthy`|healthcheck 통과 후 ⭐️ 권장|
|`service_completed_successfully`|컨테이너가 정상 종료된 후|

```
⚠️  주의:
depends_on: kafka 만 쓰면 kafka 프로세스가 완전히 뜨기 전에
producer 가 실행될 수 있음
→ healthcheck + service_healthy 조합 권장
```

---

---

# ⑤ volumes — 영구 저장소

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data   # Volume 방식
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql  # Bind Mount 방식

volumes:          # ← services: 와 같은 레벨, 파일 최하단에 선언
  postgres_data:
  kafka_data:
```

```
Volume 방식    →  postgres_data:/경로  (Docker 관리)
Bind Mount 방식 →  ./로컬경로:/컨테이너경로  (내 컴퓨터 경로 직접 연결)
```

---

---

# ⑥ networks — 컨테이너 간 통신

```yaml
services:
  kafka:
    networks:
      - train-network

  postgres:
    networks:
      - train-network

networks:         # ← services: 와 같은 레벨, 파일 최하단에 선언
  train-network:
    driver: bridge
```

```
같은 네트워크 = 컨테이너 이름으로 서로 접근 가능
kafka 컨테이너에서 → postgres:5432 로 접근
producer 컨테이너에서 → kafka:9092 로 접근
```

---

---

# ⑦ restart — 재시작 정책

```yaml
restart: no            # 재시작 안 함 (기본값)
restart: always        # 항상 재시작 (서버 재부팅 후에도)
restart: on-failure    # 에러로 종료됐을 때만 재시작
restart: unless-stopped # 직접 멈추지 않는 한 항상 재시작
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|volumes 선언 안 해서 에러|services 에만 쓰고 최상단 volumes 에 없음|최상단 `volumes:` 에 반드시 선언|
|depends_on 써도 순서 에러|`service_started` 가 기본값이라 프로세스 뜨기 전 실행|`healthcheck` + `service_healthy` 조합|
|환경변수 안 읽힘|`.env` 파일 위치가 다름|`docker-compose.yml` 과 같은 폴더에 `.env` 위치|
|컨테이너 이름으로 접근 안 됨|networks 에 같이 안 묶었음|같은 네트워크에 연결 확인|
|비밀번호 GitHub 에 올라감|`.env` 를 `.gitignore` 에 안 추가|`.gitignore` 에 `.env` 추가|

---
# 리소스 설정 — Spark Worker Lost 에러

```
증상:
  ERROR TaskSchedulerImpl: Lost executor 0 on 172.18.0.7:
  worker lost: Not receiving heartbeat for 60 seconds

원인:
  docker-compose.yml 의 SPARK_WORKER_MEMORY 가 너무 작음
  Worker 가 메모리 부족으로 응답 중단 → Executor 연결 끊김
  기본값 1G 로는 Spark Streaming 배치 반복 시 부족할 수 있음
```


```yaml
# docker-compose.yml 수정
spark-worker:
  environment:
    - SPARK_WORKER_MEMORY=2G   # 1G → 2G
    - SPARK_WORKER_CORES=2     # 1 → 2
```

```bash
# spark-submit 도 메모리 명시
spark-submit \
  --master spark://spark-master:7077 \
  --executor-memory 1g \    # 기본 512m → 1g
  --driver-memory 1g \
  --jars ... \
  consumer.py
```

```
SPARK_WORKER_MEMORY >= --executor-memory 여야 함
Worker 가 Executor 보다 커야 할당 가능

적용 순서:
  1. docker-compose.yml 수정
  2. docker compose down
  3. docker compose up -d
  4. spark-submit 메모리 옵션 추가해서 재실행
```