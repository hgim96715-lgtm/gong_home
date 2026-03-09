---
aliases:
  - 도커볼륨
  - 마운트
  - BindMount
  - NamedVolume
  - 데이터저장
  - docker_run_v
tags:
  - Docker
  - DevOps
  - Storage
  - Infrastructure
related:
  - "[[00_Docker_HomePage]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[Docker_Networking]]"
  - "[[Docker_Compose_Setup]]"
---
# 🐳 Docker_Volumes

## 개념 한 줄 요약

> **"컨테이너는 껐다 켜면 데이터가 사라진다. Volumes 은 그걸 막는 외장하드다."** 
> 컨테이너 내부에 저장된 데이터는 컨테이너가 삭제되면 함께 사라진다. 
> Volume 을 연결하면 컨테이너가 없어져도 데이터는 살아남는다.

```
컨테이너 삭제  →  내부 데이터 사라짐  ← 문제
컨테이너 삭제  →  Volume 은 유지됨   ← 해결
```

---

---

# ① Volume 의 3가지 종류

```
┌─────────────────────────────────────────────────┐
│  종류              저장 위치          주요 용도   │
├─────────────────────────────────────────────────┤
│  Volume            Docker 관리 영역   DB, 영구 데이터  │
│  Bind Mount        내 컴퓨터 경로     개발 중 코드 실시간 반영  │
│  tmpfs             메모리(RAM)        민감한 임시 데이터  │
└─────────────────────────────────────────────────┘
```

---

---

# ② Volume — Docker 가 관리하는 저장소

> **가장 권장되는 방식. Docker 가 저장 위치를 직접 관리한다.** 
> 컨테이너가 삭제돼도 Volume 은 남는다.

## docker-compose.yml 에서 사용

```yaml
services:
  postgres:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
      #  ↑ Volume 이름   ↑ 컨테이너 내부 경로

volumes:
  postgres_data:    # 여기서 Volume 을 선언해야 함
```

## docker run 에서 사용

```bash
docker run -v postgres_data:/var/lib/postgresql/data postgres:16
#              ↑ Volume 이름  ↑ 컨테이너 내부 경로
```

## Volume 명령어

```bash
# Volume 목록 확인
$ docker volume ls

# Volume 상세 정보 (실제 저장 경로 확인)
$ docker volume inspect postgres_data

# Volume 삭제
$ docker volume rm postgres_data

# 사용 안 하는 Volume 전체 삭제
$ docker volume prune
```

```
실제 저장 위치 (Mac/Linux):
/var/lib/docker/volumes/postgres_data/_data
→ Docker 가 관리하는 영역이라 직접 건드리지 않는 게 좋음
```

---

---

# ③ Bind Mount — 내 컴퓨터 경로를 직접 연결

> **내 컴퓨터의 특정 폴더를 컨테이너 안에 직접 연결한다.** 
> 코드를 수정하면 컨테이너 재시작 없이 바로 반영된다. → 개발 환경에 최적.

## docker-compose.yml 에서 사용

```yaml
services:
  app:
    image: python:3.11-slim
    volumes:
      - ./app:/app
      # ↑ 호스트 경로  ↑ 컨테이너 내부 경로
      # ./ = docker-compose.yml 이 있는 현재 폴더 기준
```

## docker run 에서 사용

```bash
docker run -v $(pwd)/app:/app python:3.11-slim
#           ↑ 절대경로 필수   ↑ 컨테이너 경로
```

```
Bind Mount 특징:
✅ 내 코드 수정 → 컨테이너 안에 즉시 반영 (개발 편의)
✅ 내 컴퓨터에서 파일 직접 확인 가능
⚠️  호스트 경로가 없으면 에러 발생
⚠️  운영 환경보다는 개발 환경에서 주로 사용
```

---

---

# ④ tmpfs — 메모리에 저장 (휘발성)

> **디스크가 아닌 RAM 에 저장한다. 컨테이너가 종료되면 무조건 사라진다.** 
> 비밀번호, 토큰 같은 민감한 임시 데이터에 사용.

```yaml
services:
  app:
    image: python:3.11-slim
    tmpfs:
      - /tmp
      - /run
```

```bash
docker run --tmpfs /tmp python:3.11-slim
```

```
tmpfs 특징:
✅ 매우 빠름 (RAM 속도)
✅ 컨테이너 종료 시 자동 삭제 → 민감 데이터 흔적 없음
❌ 컨테이너 껐다 켜면 데이터 사라짐
```

---

---

# ⑤ 세 가지 비교

| |Volume|Bind Mount|tmpfs|
|---|---|---|---|
|**저장 위치**|Docker 관리 영역|내 컴퓨터 지정 경로|메모리(RAM)|
|**영구 저장**|✅|✅|❌ (재시작 시 삭제)|
|**컨테이너 삭제 후**|유지|유지|삭제|
|**주요 용도**|DB, 영구 데이터|개발 중 코드 반영|민감한 임시 데이터|
|**권장 환경**|운영 + 개발|개발|보안 필요 시|

---

---

# ⑥ 프로젝트 적용 예시

```yaml
# 이 프로젝트(seoul-train-realtime-project) 에서의 Volume 사용

services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data   # Volume: DB 데이터 영구 저장

  kafka:
    volumes:
      - kafka_data:/var/lib/kafka/data            # Volume: Kafka 메시지 영구 저장

volumes:
  postgres_data:
  kafka_data:
```

```
컨테이너를 docker compose down 해도
postgres_data, kafka_data Volume 은 살아있음

docker compose down -v  →  Volume 까지 삭제 (초기화)
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|`docker compose down` 후 DB 데이터 사라짐|Volume 선언 안 함|`volumes:` 섹션에 Volume 선언|
|Bind Mount 경로 에러|상대경로 사용|절대경로 또는 `./` 기준 경로 사용|
|`down -v` 로 실수로 데이터 삭제|Volume 까지 삭제하는 옵션|개발 중엔 `down` 만, `-v` 는 초기화 시에만|
|Volume 이름 선언 안 함|`services` 에 쓰고 `volumes:` 에 선언 안 함|최하단 `volumes:` 섹션에 반드시 선언|

---
