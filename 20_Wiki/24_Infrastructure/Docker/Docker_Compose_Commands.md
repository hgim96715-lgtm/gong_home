---
aliases:
  - 컴포즈 명령어
  - compose up
  - compose down
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Compose_Setup]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
---
# 🐳 Dokcer Compose_Commands

## 개념 한 줄 요약

> **"docker-compose.yml 을 실행하고 관리하는 명령어 모음."**
>  `container_name` 을 지정해두면 docker 명령어에서 짧고 예측 가능한 이름으로 쓸 수 있다.

---

---

# ① container_name — 왜 지정하는가

```yaml
services:
  kafka:                          # 서비스 이름 → Compose 내부에서만 사용
    container_name: train-kafka   # 컨테이너 이름 → docker 명령어에서 사용
```

```bash
# container_name 지정했을 때 ✅
docker logs train-kafka
docker exec -it train-kafka bash
docker stop train-kafka

# container_name 없을 때 ❌ Docker 가 자동 생성
docker logs seoul-train-realtime-project-kafka-1   # 길고 예측 불가
```

```
서비스 이름 (kafka)           →  depends_on, networks 등 yml 내부에서 사용
                                  컨테이너 간 DNS 이름으로도 사용
container_name (train-kafka)  →  docker 명령어에서 사용
```

---

---

# ② docker compose up — 실행

```bash
# 기본 실행 (포그라운드 — 로그 그대로 출력)
$ docker compose up

# 백그라운드 실행 ⭐️ 주로 이걸 씀
$ docker compose up -d

# 이미지 다시 빌드하고 실행 (코드 바꿨을 때)
$ docker compose up -d --build

# 특정 서비스만 실행
$ docker compose up -d kafka
$ docker compose up -d postgres
```

---

---

# ③ docker compose down — 종료

```bash
# 컨테이너 중지 + 삭제 (Volume 은 유지)
$ docker compose down

# 컨테이너 + Volume 까지 전부 삭제 (초기화)
$ docker compose down -v

# 컨테이너 + 이미지까지 삭제
$ docker compose down --rmi all
```

```
down        →  컨테이너 삭제, Volume 유지  (데이터 살아있음)
down -v     →  컨테이너 + Volume 삭제     (데이터 전부 초기화) ⚠️
```

---

---

# ④ docker compose ps — 상태 확인

```bash
# 실행 중인 컨테이너 상태 확인
$ docker compose ps
```

```
NAME              IMAGE               STATUS
train-kafka       apache/kafka:3.7.0  running (healthy)
train-postgres    postgres:16         running (healthy)
```

```
STATUS 확인 포인트:
running (healthy)   →  정상 ✅
running (starting)  →  healthcheck 아직 통과 전
running (unhealthy) →  healthcheck 실패 ⚠️
exited              →  컨테이너 종료됨 ❌
```

>**running (starting)이 나왔다면?**
>보통 **10초~30초** 정도 기다린 후 다시 `docker compose ps`를 쳤을 때 상태가 아래와 같이 바뀌면 성공
>`Up ... (healthy)` : 모든 준비 완료! (이제 앱에서 DB나 카프카에 접속 가능) 
>`Up ... (unhealthy)` : **위험.** 설정 오류나 리소스 부족으로 서비스가 비정상 작동 중.

---

---

# ⑤ docker compose logs — 로그 확인


```bash
# 전체 서비스 로그
$ docker compose logs

# 실시간으로 계속 보기
$ docker compose logs -f

# 특정 서비스 로그만
$ docker compose logs kafka
$ docker compose logs postgres

# 특정 서비스 실시간 로그
$ docker compose logs -f kafka

# 마지막 N줄만
$ docker compose logs --tail=50 kafka
```

---

---

# ⑥ 자주 쓰는 docker 단독 명령어

> **container_name 을 지정해두면 아래 명령어가 편해진다.**

```bash
# 컨테이너 내부로 들어가기
$ docker exec -it train-kafka bash
$ docker exec -it train-postgres bash

# PostgreSQL 직접 접속
$ docker exec -it train-postgres psql -U train_user -d train_db

# Kafka 토픽 확인
$ docker exec train-kafka /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --list

# 실행 중인 컨테이너 전체 목록
$ docker ps

# 컨테이너 개별 로그
$ docker logs train-kafka
$ docker logs -f train-postgres   # 실시간
```

---

---

# ⑦ 전체 흐름 — 프로젝트 시작부터 종료까지(예시)

```bash
# 1. 프로젝트 폴더로 이동
$ cd <프로젝트 폴더명>

# 2. 컨테이너 실행
$ docker compose up -d

# 3. 상태 확인
$ docker compose ps

# 4. 로그 확인
$ docker compose logs -f

# 5. 컨테이너 내부 접속
$ docker exec -it <container_name> <실행할 명령어>

# 예시
$ docker exec -it <container_name> bash                         # 일반 bash 접속
$ docker exec -it <container_name> psql -U <유저명> -d <DB명>   # PostgreSQL 접속

# 6. 작업 끝나면 종료 (데이터 유지)
$ docker compose down

# 7. 초기화할 때만 (데이터 삭제)
$ docker compose down -v
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|코드 바꿨는데 반영 안 됨|이미지 재빌드 안 함|`docker compose up -d --build`|
|로그가 안 보임|`-d` 백그라운드로 실행|`docker compose logs -f`|
|컨테이너 이름이 너무 길어|`container_name` 미지정|`container_name` 직접 지정|
|down 후 데이터 사라짐|`down -v` 로 Volume 까지 삭제|데이터 유지할 땐 `-v` 없이 `down`|
|healthy 가 안 뜸|healthcheck 설정 안 함|`healthcheck` 추가 후 `depends_on` 에 `service_healthy`|

---
