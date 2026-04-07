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

# ① Docker 환경 구성

## docker-compose.yml

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16
    container_name: dbt-postgres
    environment:
      POSTGRES_USER: dbt_user
      POSTGRES_PASSWORD: dbt_password
      POSTGRES_DB: dbt_db
    ports:
      - "5435:5432"    # 다른 프로젝트와 포트 충돌 방지
    volumes:
      - dbt_postgres_data:/var/lib/postgresql/data

  dbt:
    image: python:3.11-slim
    container_name: dbt-runner
    working_dir: /dbt
    volumes:
      - ./dbt_project:/dbt          # 로컬 프로젝트 마운트
      - ./profiles:/root/.dbt       # profiles.yml 마운트
    depends_on:
      - postgres
    command: tail -f /dev/null      # 컨테이너 유지

volumes:
  dbt_postgres_data:
```

## 컨테이너 실행

```bash
docker compose up -d
docker compose ps    # 상태 확인
```

---

---

# ② dbt 설치

## 컨테이너 안에서 설치

```bash
# dbt 컨테이너 접속
docker exec -it dbt-runner bash

# dbt-core + PostgreSQL 어댑터 설치
pip install dbt-core dbt-postgres

# 설치 확인
dbt --version
```

## 로컬에서 설치 (컨테이너 없이)

```bash
pip install dbt-core dbt-postgres

dbt --version
# Core:      1.x.x
# Plugins:   postgres: 1.x.x
```

```
어댑터는 DB 종류에 따라 다름:
  dbt-postgres    PostgreSQL
  dbt-snowflake   Snowflake
  dbt-bigquery    BigQuery
  dbt-redshift    Redshift
```

---

---

# ③ 프로젝트 초기화

```bash
# dbt 컨테이너 안에서
dbt init 프로젝트명

# 예시
dbt init my_dbt_project
```

```
초기화 시 물어보는 것:
  1. DB 종류 선택 → postgres 입력
  2. 이후 설정은 profiles.yml 에서 직접 수정
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

```yaml
# ~/.dbt/profiles.yml

my_dbt_project:          # dbt_project.yml 의 profile 과 이름 일치해야 함
  target: dev            # 기본 실행 환경
  outputs:
    dev:                 # 개발 환경
      type: postgres
      host: postgres     # docker-compose 서비스명 (컨테이너 안)
      # host: localhost  # 로컬에서 실행 시
      port: 5432         # 컨테이너 내부 포트
      # port: 5435       # 로컬에서 실행 시 (매핑 포트)
      user: dbt_user
      password: dbt_password
      dbname: dbt_db
      schema: public     # 모델이 생성될 스키마
      threads: 4         # 병렬 실행 스레드 수

    prod:                # 운영 환경 (필요 시)
      type: postgres
      host: prod-host
      port: 5432
      user: prod_user
      password: "{{ env_var('DBT_PROD_PASSWORD') }}"  # 환경변수 권장
      dbname: prod_db
      schema: analytics
      threads: 8
```

```
host 주의:
  dbt 컨테이너 → postgres 컨테이너 통신
  → host: postgres  (docker-compose 서비스명)

  로컬 터미널 → postgres 컨테이너 통신
  → host: localhost / port: 5435 (매핑 포트)
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