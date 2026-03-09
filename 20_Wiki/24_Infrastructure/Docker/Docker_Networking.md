---
aliases:
  - 도커 네트워킹
  - 컨테이너 통신
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Compose_Setup]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
---

# 🐳 Docker_Networking

## 개념 한 줄 요약

> **"컨테이너는 기본적으로 서로 모르는 섬이다. 네트워크는 그 섬들을 이어주는 다리다."** 
> 같은 네트워크에 속한 컨테이너끼리는 컨테이너 이름으로 서로를 찾을 수 있다.

```
컨테이너 A  ──(같은 네트워크)──  컨테이너 B
  kafka                   producer
    ↑                        ↑
  컨테이너 이름 = DNS 주소로 사용 가능
  producer → kafka:9092 로 접속 가능
```

---

---

# ① 네트워크 드라이버 종류

```
bridge  →  기본값. 컨테이너끼리 같은 가상 네트워크 안에서 통신
host    →  컨테이너가 내 컴퓨터 네트워크를 그대로 사용
none    →  네트워크 완전 차단 (격리)
```

---

---

# ② bridge — 기본 네트워크

> **Docker 의 기본 네트워크 드라이버.** 
> 같은 bridge 네트워크 안에 있는 컨테이너끼리만 통신 가능.

## 기본 bridge (자동 생성)

```bash
# 네트워크 지정 없이 컨테이너 실행 → 기본 bridge 에 연결
$ docker run -d --name kafka apache/kafka:3.7.0
$ docker run -d --name producer my-producer
```

```
⚠️  기본 bridge 의 문제점:
컨테이너 이름으로 DNS 조회가 안 됨
producer 컨테이너에서 kafka 라는 이름으로 접근 불가
IP 주소를 직접 써야 함 → 컨테이너 재시작하면 IP 바뀜 → 불편
```

## 사용자 정의 bridge (권장) ⭐️

```bash
# 네트워크 직접 생성
$ docker network create train-network

# 같은 네트워크로 컨테이너 실행
$ docker run -d --name kafka --network train-network apache/kafka:3.7.0
$ docker run -d --name producer --network train-network my-producer
```

```
✅ 사용자 정의 bridge 의 장점:
컨테이너 이름 = DNS 주소
producer 컨테이너에서 kafka:9092 로 바로 접근 가능
IP 가 바뀌어도 이름으로 찾아가니까 상관없음
```

---

---

# ③ 컨테이너 간 DNS — 이름으로 서로를 찾는다

> **같은 네트워크 안에서 컨테이너 이름이 곧 호스트명이 된다.**

```
네트워크: train-network

컨테이너: train-kafka   → kafka   라는 이름으로 접근 가능
컨테이너: train-postgres → postgres 라는 이름으로 접근 가능
```

```python
# producer 코드에서 kafka 에 접속할 때
# IP 주소 ❌
producer = KafkaProducer(bootstrap_servers='172.18.0.2:9092')

# 컨테이너 이름 ✅
producer = KafkaProducer(bootstrap_servers='kafka:9092')
#                                            ↑ 컨테이너 이름이 DNS 주소
```

```yaml
# docker-compose.yml 에서 PostgreSQL 접속
# IP 주소 ❌
DATABASE_URL: postgresql://user:pass@172.18.0.3:5432/db

# 컨테이너 이름 ✅
DATABASE_URL: postgresql://user:pass@postgres:5432/db
```

---

---

# ④ -p 포트포워딩 — 외부에서 컨테이너로 들어오기

> **내 컴퓨터(호스트)의 포트와 컨테이너의 포트를 연결한다.** 브라우저나 외부 앱에서 컨테이너 안으로 접근할 때 사용.

```bash
docker run -p 호스트포트:컨테이너포트
#             ↑ 내 컴퓨터    ↑ 컨테이너 안
```

```bash
# 예시
docker run -p 5432:5432 postgres:16
#              ↑ 내 컴퓨터 5432  ↑ 컨테이너 안 5432

docker run -p 9092:9092 apache/kafka:3.7.0
docker run -p 8088:8088 apache/superset
```

```
접근 흐름:

브라우저/DBeaver → localhost:5432 → 컨테이너:5432 → PostgreSQL

포트포워딩 없으면:
브라우저 → localhost:5432 → 컨테이너에 접근 불가 ❌
```

## 컨테이너 간 통신 vs 외부 접근

```
컨테이너끼리 통신  →  포트포워딩 불필요, 컨테이너 이름으로 접근
외부(브라우저 등)  →  포트포워딩 필수, localhost:포트 로 접근
```

---

---

# ⑤ host — 컨테이너가 내 컴퓨터 네트워크를 그대로 사용

> **컨테이너가 별도의 네트워크 없이 내 컴퓨터 네트워크를 직접 사용한다.** 포트포워딩 없이 내 컴퓨터 포트를 바로 사용.

```bash
docker run --network host apache/kafka:3.7.0
# 컨테이너가 쓰는 9092 포트 = 내 컴퓨터의 9092 포트
# -p 옵션 불필요
```

```
⚠️  host 네트워크 주의사항:
Mac / Windows 에서는 host 네트워크 지원 안 함 (Linux 전용)
포트 충돌 위험 (컨테이너가 내 컴퓨터 포트를 직접 점유)
개발 환경보다 운영 서버에서 성능 최적화 목적으로 사용
```

---

---

# ⑥ Docker Compose 에서 네트워크

> **Docker Compose 는 자동으로 하나의 네트워크를 만들고 모든 컨테이너를 연결한다.** 
> 따로 네트워크를 설정하지 않아도 같은 `docker-compose.yml` 안의 컨테이너끼리는 이름으로 통신 가능.

```yaml
# 네트워크 별도 설정 없어도 동작
services:
  kafka:
    image: apache/kafka:3.7.0
    container_name: train-kafka

  postgres:
    image: postgres:16
    container_name: train-postgres

# → kafka, postgres 컨테이너는 자동으로 같은 네트워크에 연결됨
# → 코드에서 kafka:9092, postgres:5432 로 바로 접근 가능
```

## 명시적으로 네트워크를 지정하고 싶을 때

```yaml
services:
  kafka:
    image: apache/kafka:3.7.0
    networks:
      - train-network

  postgres:
    image: postgres:16
    networks:
      - train-network

networks:
  train-network:
    driver: bridge
```

---

---

# ⑦ 네트워크 명령어

```bash
# 네트워크 목록
$ docker network ls

# 네트워크 상세 정보 (어떤 컨테이너가 연결됐는지)
$ docker network inspect train-network

# 네트워크 생성
$ docker network create train-network

# 실행 중인 컨테이너를 네트워크에 연결
$ docker network connect train-network train-kafka

# 네트워크에서 컨테이너 분리
$ docker network disconnect train-network train-kafka

# 사용 안 하는 네트워크 삭제
$ docker network prune
```

---

---

# 전체 흐름 정리

```
외부(브라우저/DBeaver/터미널)
        ↓ localhost:9092
    포트포워딩 (-p)
        ↓
  ┌─────────────────────────────┐
  │       train-network         │
  │                             │
  │  train-kafka:9092           │
  │       ↕ 컨테이너 이름으로    │
  │  train-postgres:5432        │
  │       ↕ 통신 가능            │
  │  train-producer             │
  └─────────────────────────────┘
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|컨테이너 이름으로 접근이 안 됨|기본 bridge 사용|사용자 정의 네트워크 또는 Compose 사용|
|`localhost` 로 접근이 안 됨|포트포워딩 안 함|`-p 호스트:컨테이너` 추가|
|코드에서 IP 주소 하드코딩|재시작 시 IP 바뀜|컨테이너 이름으로 접근|
|Mac 에서 `--network host` 안 됨|Linux 전용 기능|bridge + 포트포워딩으로 대체|

---
