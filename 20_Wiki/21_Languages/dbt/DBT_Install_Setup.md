---
aliases:
  - dbt 설치
  - dbt 설정
  - profiles.yml
tags:
  - dbt
related:
  - "[[00_DBT_HomePage]]"
  - "[[DBT_Overview]]"
  - "[[DBT_Project_Structure]]"
---


# DBT_Install_Setup — 설치 & 환경 구성

## 한 줄 요약

```
dbt-core + dbt-postgres 설치
profiles.yml 로 PostgreSQL 연결 설정
dbt init 으로 프로젝트 생성
```

---

---

# ① Docker 환경 구성 ⭐️

## 문제 — docker compose down 하면 dbt 사라짐

```
image: python:3.11-slim 방식:
  컨테이너가 내려가면 pip install 한 것 전부 사라짐
  docker compose up 할 때마다 dbt 재설치 필요 → 번거로움

해결:
  Dockerfile 로 dbt 가 이미 설치된 이미지를 직접 빌드
  docker compose down / up 해도 dbt 유지됨
```

## Dockerfile 작성

```dockerfile
# Dockerfile

FROM python:3.11-slim

# git + dbt-postgres 설치 (이미지 안에 포함)
RUN apt-get update && apt-get install -y git \
    && pip install --no-cache-dir dbt-postgres

WORKDIR /dbt_exhibition
```

```
FROM python:3.11-slim:
  경량 파이썬 이미지 베이스

RUN apt-get install git:
  dbt 패키지 설치 시 git 필요

pip install dbt-postgres:
  dbt-core + PostgreSQL 어댑터 한 번에 설치
  이미지 빌드 시 1번만 실행됨

WORKDIR /dbt_exhibition:
  컨테이너 안 기본 작업 디렉토리
```

## docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:16
    container_name: exhibition-postgres
    environment:
      POSTGRES_USER: exhibition_user
      POSTGRES_PASSWORD: exhibition_password
      POSTGRES_DB: exhibition_db
    ports:
      - "5435:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - exhibition-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U exhibition_user -d exhibition_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  dbt:
    build: .                        # ← Dockerfile 로 빌드 (image 대신)
    container_name: exhibition-dbt
    working_dir: /dbt_exhibition
    volumes:
      - ./dbt_exhibition:/dbt_exhibition   # 로컬 프로젝트 마운트
    ports:
      - "8585:8585"                 # dbt docs serve 포트
    networks:
      - exhibition-network
    depends_on:
      postgres:
        condition: service_healthy  # postgres 준비 완료 후 시작
    command: tail -f /dev/null      # 컨테이너 유지, 필수!!

networks:
  exhibition-network:

volumes:
  postgres_data:
```

```
image: vs build:
  image: python:3.11-slim  → DockerHub 에서 그대로 가져옴
                              매번 pip install 필요
  build: .                 → Dockerfile 로 직접 빌드
                              dbt 포함된 이미지 생성
                              down/up 해도 재설치 불필요
```

## 빌드 & 실행

```bash
# 처음 실행 (이미지 빌드 포함)
docker compose up -d --build

# 이후 실행 (빌드 없이)
docker compose up -d

# 상태 확인
docker compose ps

# dbt 컨테이너 접속
docker exec -it exhibition-dbt bash

# 설치 확인
dbt --version
```

```
--build:
  Dockerfile 변경 시 이미지 재빌드
  처음 실행 시 / Dockerfile 수정 시만 필요
  평소엔 그냥 docker compose up -d
```

---

---

# ② dbt 설치 확인

```bash
# 컨테이너 접속 후 확인
docker exec -it exhibition-dbt bash

dbt --version
# Core:      1.x.x
# Plugins:   postgres: 1.x.x
```

```
Dockerfile 빌드 방식이면:
  docker compose down 후 up 해도 dbt 유지
  매번 pip install 불필요

어댑터 종류:
  dbt-postgres    PostgreSQL
  dbt-snowflake   Snowflake
  dbt-bigquery    BigQuery
  dbt-redshift    Redshift
```

---

---

# ③ 프로젝트 초기화

```
컨테이너 안에서 dbt init 실행
반드시 docker exec 으로 컨테이너 접속 후 실행
```

## 컨테이너 접속 → dbt init

```bash
# 1. dbt 컨테이너 안으로 접속
docker exec -it dbt-runner bash

# 2. 프로젝트 초기화
dbt init exhibition_project
```

## dbt init 대화형 진행 과정 ⭐️

```bash
dbt init exhibition_project

# Which database would you like to use?
# [1] postgres
# [2] redshift
# ...
# Enter a number: 1          ← postgres 번호 입력

# host (hostname for the postgres instance):
postgres                     ← docker-compose 서비스명 입력
                             ← 로컬이면 localhost

# port [5432]:
                             ← 그냥 Enter (기본값 5432 사용)

# user (dev username):
dbt_user                     ← DB 설정에 맞게 입력

# pass (dev password):
dbt_password

# dbname (default database that dbt will build objects in):
dbt_db

# schema (default schema that dbt will build objects in):
public                       ← 기본 스키마

# threads (1 or more) [1]:
4                            ← 병렬 스레드 수
```

```
이 과정에서 입력한 내용이
자동으로 ~/.dbt/profiles.yml 에 저장됨

→ 나중에 수정하려면 profiles.yml 직접 편집
```

## 생성되는 폴더 구조

```
my_dbt_project/
├── dbt_project.yml      # 프로젝트 설정 파일
├── models/              # SQL 변환 모델
│   └── example/
│       ├── my_first_dbt_model.sql
│       └── schema.yml
├── tests/               # 커스텀 테스트
├── seeds/               # CSV 파일 → DB 테이블
├── snapshots/           # SCD Type 2 이력 관리
├── macros/              # 재사용 가능한 SQL 함수
├── analyses/            # 분석용 SQL (실행 안 함)
└── target/              # 컴파일된 SQL (자동 생성)
```

---

---

# ④ profiles.yml — DB 연결 설정 ⭐️

```
~/.dbt/profiles.yml  ← 기본 위치
프로젝트와 분리 저장 (보안 / 재사용)
DB 접속 정보를 여기에 설정
```

## 폴더 생성

```bash
mkdir -p ~/.dbt
```

## profiles.yml 작성

```
dbt init 실행하면 자동으로 생성됨
나중에 수정하려면 직접 편집
```

```bash
# 위치 확인
cat ~/.dbt/profiles.yml

# 직접 편집
vim ~/.dbt/profiles.yml
nano ~/.dbt/profiles.yml
```

## 각 필드에 뭘 써야 하는가 ⭐️

```yaml
# ~/.dbt/profiles.yml

exhibition:                  # ← dbt_project.yml 의 profile: 값과 일치해야 함
  target: dev                # ← 기본 실행 환경 (dev 로 고정)
  outputs:
    dev:
      type: postgres         # ← DB 종류 (고정)
      host: postgres         # ← docker-compose 서비스명 (컨테이너 이름)
      port: 5432             # ← 컨테이너 내부 포트 (5432 고정)
      user: exhibition_user  # ← docker-compose 의 POSTGRES_USER
      pass: exhibition_password  # ← docker-compose 의 POSTGRES_PASSWORD
      dbname: exhibition_db  # ← docker-compose 의 POSTGRES_DB
      schema: public         # ← 모델이 생성될 스키마 (public 으로 시작)
      threads: 4             # ← 병렬 실행 스레드 수
```

## docker-compose.yml ↔ profiles.yml 연결 관계 ⭐️

```yaml
# docker-compose.yml
services:
  postgres:
    container_name: exhibition-postgres   ← host 에 쓰는 이름
    environment:
      POSTGRES_USER: exhibition_user      ← user
      POSTGRES_PASSWORD: exhibition_password  ← pass
      POSTGRES_DB: exhibition_db          ← dbname
    ports:
      - "5435:5432"                       ← 로컬에서 접속 시 5435

  dbt:
    container_name: exhibition-dbt
```

```
profiles.yml 각 값의 출처:

  host     → postgres 컨테이너의 서비스명 (docker-compose 에서 services: 아래 이름)
             컨테이너 안에서 실행 → 서비스명 사용 (postgres / exhibition-postgres 등)
             로컬 터미널에서 실행 → localhost

  port     → 컨테이너 안에서 실행 → 5432 (내부 포트)
             로컬 터미널에서 실행 → 5435 (매핑 포트)

  user     → docker-compose 의 POSTGRES_USER 값
  pass     → docker-compose 의 POSTGRES_PASSWORD 값
  dbname   → docker-compose 의 POSTGRES_DB 값
  schema   → public 으로 시작 / 필요에 따라 변경
  threads  → 병렬 실행 수 (보통 4)
```

## password vs pass 주의

```yaml
# PostgreSQL 어댑터에서는 pass 또는 password 둘 다 사용 가능
# dbt init 으로 생성하면 보통 pass 로 생성됨

dev:
  pass: exhibition_password     # dbt init 자동 생성 시
  # password: exhibition_password  # 수동 작성 시 둘 다 가능
```

## 실전 예시 — exhibition 프로젝트

```yaml
# ~/.dbt/profiles.yml

exhibition:
  target: dev
  outputs:
    dev:
      type: postgres
      host: postgres          # docker-compose 서비스명
      port: 5432              # 컨테이너 내부 포트
      user: exhibition_user
      pass: exhibition_password
      dbname: exhibition_db
      schema: public
      threads: 4
```

```
host 주의:
  dbt 컨테이너 안에서 실행 → host: postgres (서비스명)
  로컬 터미널에서 실행     → host: localhost, port: 5435

  헷갈리면:
    dbt debug 실행 → Connection test: OK 면 성공
    Connection refused 면 host / port 확인
```

## prod 환경 — 언제 필요한가 ⭐️

```
지금 당장은 dev 환경만으로 충분
아래 상황이 되면 prod 환경 추가

prod 환경이 필요한 시점:
  팀 협업 시작 → dev 는 개인 / prod 는 공용 DB 분리
  실제 서비스 배포 → 분석가 / BI 도구가 사용하는 DB 따로
  CI/CD 파이프라인 연결 → GitHub Actions / Airflow 에서 prod 로 배포
  스키마 분리 필요 → dev 는 public / prod 는 analytics
```

```
dev vs prod 차이:

  dev:
    개발 / 테스트 용도
    실수해도 괜찮음 (내 로컬 / 테스트 DB)
    dbt run 기본값

  prod:
    실제 분석가 / BI 도구가 쓰는 데이터
    실수하면 안 됨
    dbt run --target prod 로 명시적으로 실행
```

```bash
# dev 실행 (기본값 = target: dev)
dbt run

# prod 실행 (명시적으로 target 지정)
dbt run --target prod
```

```yaml
exhibition:
  target: dev              # ← 기본값은 dev
  outputs:
    dev:
      type: postgres
      host: postgres        # 로컬 Docker
      port: 5432
      user: exhibition_user
      pass: exhibition_password
      dbname: exhibition_db
      schema: public
      threads: 4

    prod:                   # ← 팀 공용 / 실제 서비스 DB
      type: postgres
      host: prod-host       # 실제 서버 주소
      port: 5432
      user: prod_user
      pass: "{{ env_var('DBT_PROD_PASSWORD') }}"  # 비밀번호는 환경변수로
      dbname: prod_db
      schema: analytics     # dev 와 다른 스키마로 분리
      threads: 8
```

```
처음 시작할 때:
  dev 만 써도 충분
  prod 는 필요해질 때 추가
  지금은 dev 완성에 집중
```

## dbt_project.yml 에서 profile 연결

```yaml
# dbt_project.yml
name: 'my_dbt_project'
version: '1.0.0'
config-version: 2

profile: 'my_dbt_project'   # profiles.yml 이름과 일치

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]

models:
  my_dbt_project:
    +materialized: view    # 기본 materialization
```

---

---

# ⑤ 연결 테스트

```bash
# dbt 컨테이너 안에서
cd /dbt/my_dbt_project

dbt debug
```

```
정상 출력:
  Connection:
    host: postgres
    port: 5432
    user: dbt_user
    database: dbt_db
    schema: public
    ...
  Connection test: OK connection ok
```

```
에러 유형:
  Connection refused  → postgres 컨테이너 미실행 / host 잘못됨
  password failed     → 비밀번호 틀림
  database not found  → dbname 틀림
  profile not found   → profiles.yml 위치 또는 이름 확인
```

---

---

# ⑥ 첫 실행

```bash
# 예시 모델 실행
dbt run

# 테스트 실행
dbt test

# 문서 생성 + 서버
dbt docs generate
dbt docs serve --port 8081   # localhost:8081 에서 확인
```

---

---

# 폴더 구조 한눈에

```
프로젝트/
├── docker-compose.yml
├── profiles/
│   └── profiles.yml          # ~/.dbt 대신 볼륨 마운트
└── dbt_project/
    ├── dbt_project.yml
    ├── models/
    ├── seeds/
    ├── tests/
    └── macros/
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`profile not found`|profiles.yml 이름 불일치|dbt_project.yml 의 `profile:` 값과 동일하게|
|`Connection refused`|host 잘못됨|컨테이너 안 → `postgres` / 로컬 → `localhost`|
|포트 에러|내부/외부 포트 혼동|컨테이너 안 → `5432` / 로컬 → `5435`|
|`schema does not exist`|schema 없음|PostgreSQL 에서 `CREATE SCHEMA 스키마명`|
|`dbt: command not found`|설치 안 됨|`pip install dbt-core dbt-postgres`|
|`docker compose down` 후 dbt 사라짐|`image:` 방식 사용|`Dockerfile` + `build: .` 방식으로 변경|
|이미지 변경 반영 안 됨|재빌드 안 함|`docker compose up -d --build`|