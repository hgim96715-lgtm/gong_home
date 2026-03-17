---
aliases:
  - docker run
  - docker start
  - docker restart
  - docker stop
  - docker rm
  - docker ps
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Image_vs_Container]]"
  - "[[Docker_Container_Interaction]]"
  - "[[Docker_Compose_Commands]]"
---
# Docker_Compose_Commands — Compose 실행 명령어

## 한 줄 요약

```
docker-compose.yml 을 실행하고 관리하는 명령어 모음
container_name 을 지정해두면 docker 명령어에서
짧고 예측 가능한 이름으로 쓸 수 있다
```

---

---

# ① container_name — 왜 지정하는가

```yaml
services:
  kafka:                          # 서비스 이름 → yml 내부 / 컨테이너 간 DNS
    container_name: train-kafka   # 컨테이너 이름 → docker 명령어에서 사용
```

```bash
# container_name 지정했을 때 ✅
docker logs train-kafka
docker exec -it train-kafka bash
docker stop train-kafka

# container_name 없을 때 → Docker 가 자동 생성 ❌
docker logs seoul-train-realtime-project-kafka-1   # 길고 예측 불가
```

```
서비스 이름 (kafka)          → depends_on / networks 등 yml 내부
                               컨테이너 간 DNS 이름으로도 사용
container_name (train-kafka) → docker 명령어에서 사용
```

---

---

# ② docker compose up — 실행

```bash
# 백그라운드 실행 ← 주로 이걸 씀
docker compose up -d

# 포그라운드 (로그 그대로 출력)
docker compose up

# 이미지 다시 빌드 후 실행 (코드 바꿨을 때)
docker compose up -d --build

# 특정 서비스만 실행
docker compose up -d kafka
docker compose up -d postgres
```

---

---

# ③ docker compose down — 종료

```bash
# 컨테이너 중지 + 삭제 (Volume 유지 → 데이터 살아있음)
docker compose down

# 컨테이너 + Volume 까지 삭제 (초기화)
docker compose down -v

# 컨테이너 + 이미지까지 삭제
docker compose down --rmi all
```

```
⚠️ down vs down -v:
  down      Volume 유지 → DB 데이터 그대로
  down -v   Volume 삭제 → 데이터 전부 사라짐

  init.sql 을 수정했을 때:
  init.sql 은 컨테이너 최초 실행 시에만 자동 실행
  볼륨이 이미 있으면 실행 안 됨
  → docker compose down -v 후 up 해야 반영됨
```

---

---

# ④ docker compose ps — 상태 확인

```bash
docker compose ps
```

```
NAME              IMAGE               STATUS
train-kafka       apache/kafka:3.7.0  running (healthy)
train-postgres    postgres:16         running (healthy)
```

```
STATUS 의미:
  running (healthy)    정상 ✅
  running (starting)   healthcheck 아직 통과 전 (10~30초 기다리면 됨)
  running (unhealthy)  healthcheck 실패 ⚠️ 설정 오류 확인
  exited               컨테이너 종료됨 ❌ 로그 확인 필요
```

---

---

# ⑤ docker compose logs — 로그 확인

```bash
# 전체 서비스 로그
docker compose logs

# 실시간으로 계속 보기
docker compose logs -f

# 특정 서비스만
docker compose logs kafka
docker compose logs -f kafka      # 특정 서비스 실시간

# 마지막 N줄만
docker compose logs --tail 50 kafka
```

> 컨테이너 개별 로그 → [[Docker_Container_Interaction]] 참고 `docker logs -f train-kafka` / `docker exec -it ...` 등

---

---

# ⑥ 전체 흐름 — 시작부터 종료까지

```bash
# 1. 프로젝트 폴더로 이동
cd hospital-project

# 2. 컨테이너 실행
docker compose up -d

# 3. 상태 확인
docker compose ps

# 4. 로그 확인
docker compose logs -f

# 5. 컨테이너 내부 접속
docker exec -it hospital-postgres bash
docker exec -it hospital-postgres psql -U hospital_user -d hospital_db

# 6. 작업 끝나면 종료 (데이터 유지)
docker compose down

# 7. 초기화 (데이터 삭제 후 처음부터)
docker compose down -v && docker compose up -d
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|코드 바꿨는데 반영 안 됨|이미지 재빌드 안 함|`docker compose up -d --build`|
|로그가 안 보임|`-d` 백그라운드로 실행|`docker compose logs -f`|
|컨테이너 이름이 너무 길어|`container_name` 미지정|`container_name` 직접 지정|
|down 후 데이터 사라짐|`down -v` 로 Volume 삭제|데이터 유지할 땐 `-v` 없이 `down`|
|init.sql 수정했는데 반영 안 됨|볼륨이 이미 있어서 미실행|`docker compose down -v` 후 재시작|
|healthy 가 안 뜸|healthcheck 실패|로그 확인 후 설정 점검|