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
# Dockerfile (프로젝트 루트에 위치)

FROM python:3.11-slim

RUN apt-get update && apt-get install -y git \
    && pip install --no-cache-dir dbt-postgres

WORKDIR /dbt
```

```
FROM python:3.11-slim  → 경량 파이썬 이미지
git 설치              → dbt 패키지 설치 시 필요
pip install dbt-postgres → dbt-core + PostgreSQL 어댑터
WORKDIR /dbt          → 컨테이너 기본 작업 디렉토리
                         ← /dbt 로 설정해야 dbt init 이중 폴더 방지
```

## docker-compose.yml dbt 서비스

```yaml
dbt:
  build: .                          # Dockerfile 로 빌드
  container_name: exhibition-dbt
  working_dir: /dbt                 # ✅ /dbt (상위 폴더)
  volumes:
    - ./dbt:/dbt                    # ✅ ./dbt → /dbt 마운트
  ports:
    - "8585:8585"
  networks:
    - exhibition-network
  depends_on:
    postgres:
      condition: service_healthy
  command: tail -f /dev/null        # 컨테이너 유지 필수

# Airflow 에서도 dbt 경로 일치
airflow:
  volumes:
    - ./dbt:/opt/airflow/dbt        # ✅ ./dbt 마운트
```

## ⚠️ 볼륨 경로 — 이중 폴더 발생하는 실수

```
❌ 잘못된 방식
  volumes:
    - ./dbt_exhibition:/dbt_exhibition
  working_dir: /dbt_exhibition

  → 컨테이너 안에서 dbt init dbt_exhibition 실행하면
  → /dbt_exhibition/dbt_exhibition/ 이중 폴더 생성

✅ 올바른 방식
  volumes:
    - ./dbt:/dbt
  working_dir: /dbt

  → dbt init dbt_exhibition 실행하면
  → /dbt/dbt_exhibition/ 생성
  → 로컬의 ./dbt/dbt_exhibition/ 으로 정상 매핑
```

```
로컬 폴더 구조:
  exhibition-project/
    ├── dbt/                        ← 볼륨 마운트 대상
    │   └── dbt_exhibition/         ← dbt init 으로 생성됨
    │       ├── dbt_project.yml
    │       └── models/
    ├── postgres/
    ├── crawl/
    └── docker-compose.yml
```

## 빌드 & 실행

```bash
# 처음 실행 (이미지 빌드 포함)
docker compose up -d --build

# 이후 실행
docker compose up -d

# dbt 컨테이너 접속
docker exec -it exhibition-dbt bash

# 설치 확인
dbt --version
# Core:      1.x.x
# Plugins:   postgres: 1.x.x
```

```
--build:
  Dockerfile 변경 시 이미지 재빌드
  처음 실행 시 / Dockerfile 수정 시만 필요
  평소엔 docker compose up -d 만
```

---

---

# ② 프로젝트 초기화 — dbt init ⭐️

```bash
# 1. dbt 컨테이너 접속
docker exec -it exhibition-dbt bash

# 2. 작업 디렉토리 확인
pwd
# /dbt   ← WORKDIR 설정값

# 3. 프로젝트 초기화
dbt init dbt_exhibition
```

## dbt init 대화형 진행

```bash
dbt init dbt_exhibition

# Which database would you like to use?
# [1] postgres
# Enter a number: 1

# host: postgres           ← docker-compose 서비스명
# port [5432]:             ← Enter (기본값 5432)
# user: exhibition_user
# pass: exhibition_password
# dbname: exhibition_db
# schema: public
# threads [1]: 4
```

```
입력 내용이 ~/.dbt/profiles.yml 에 자동 저장됨
나중에 수정하려면 profiles.yml 직접 편집
```

## 생성되는 폴더 구조

```
/dbt/                              # working_dir (컨테이너)
└── dbt_exhibition/                # dbt init 으로 생성
    ├── dbt_project.yml            # 프로젝트 설정
    ├── models/                    # SQL 변환 모델
    │   ├── staging/               # 정제 레이어
    │   │   ├── sources.yml
    │   │   ├── schema.yml
    │   │   └── stg_exhibitions.sql
    │   └── marts/                 # 분석 레이어
    │       ├── schema.yml
    │       ├── mart_exhibition_summary.sql
    │       ├── mart_location_analysis.sql
    │       ├── mart_price_analysis.sql
    │       └── mart_current_exhibitions.sql
    ├── tests/
    ├── seeds/
    ├── macros/
    └── target/                    # 컴파일 결과 (자동 생성)
```

---

---

# ③ profiles.yml — DB 연결 설정 ⭐️

```
~/.dbt/profiles.yml  ← 기본 위치
dbt init 실행 시 자동 생성
DB 접속 정보를 여기에 설정
```

## 각 필드 출처

```yaml
# ~/.dbt/profiles.yml

dbt_exhibition:           # dbt_project.yml 의 profile: 값과 일치해야 함
  target: dev
  outputs:
    dev:
      type: postgres
      host: postgres      # docker-compose 서비스명 (컨테이너 안에서 실행 시)
      port: 5432          # 컨테이너 내부 포트
      user: exhibition_user
      pass: exhibition_password
      dbname: exhibition_db
      schema: public
      threads: 4
```

## .env 파일 사용 시 — env_var() 방식 ⭐️

```
비밀번호를 profiles.yml 에 직접 쓰지 않고
.env 파일의 환경변수를 참조하는 방식
→ profiles.yml 을 git 에 올려도 비밀번호 노출 없음
```

```yaml
# ~/.dbt/profiles.yml

dbt_exhibition:
  target: dev
  outputs:
    dev:
      type: postgres
      host: postgres
      port: 5432
      dbname: exhibition_db
      user: "{{ env_var('POSTGRES_USER') }}"         # .env 의 POSTGRES_USER
      password: "{{ env_var('POSTGRES_PASSWORD') }}" # .env 의 POSTGRES_PASSWORD
      schema: public
      threads: 4
```

```bash
# .env
POSTGRES_USER=exhibition_user
POSTGRES_PASSWORD=exhibition_password
POSTGRES_DB=exhibition_db
```

```
pass vs password:
  pass     → dbt init 자동 생성 시 기본값
  password → 수동 작성 시 사용 가능 (둘 다 인식됨)
  env_var() 쓸 때는 password 로 쓰는 게 일반적

env_var() 동작:
  dbt 실행 시 환경변수에서 값을 읽어옴
  docker-compose.yml 의 env_file: .env 설정으로
  컨테이너 안에 환경변수가 주입되어 있어야 함
```

```yaml
# docker-compose.yml dbt 서비스에 env_file 추가 필요
dbt:
  build: .
  env_file:
    - .env                  # ← .env 를 컨테이너에 주입
  working_dir: /dbt
  volumes:
    - ./dbt:/dbt
  ...
```

```
직접 입력 vs env_var() 비교:

  직접 입력:
    user: exhibition_user
    pass: exhibition_password
    → 간단하지만 profiles.yml 에 비밀번호 노출
    → git 에 올리면 안 됨

  env_var():
    user: "{{ env_var('POSTGRES_USER') }}"
    password: "{{ env_var('POSTGRES_PASSWORD') }}"
    → .env 파일에서 읽어옴
    → profiles.yml 을 git 에 올려도 안전
    → .env 는 .gitignore 에 추가
```

## docker-compose.yml ↔ profiles.yml 연결 관계

```yaml
# docker-compose.yml
postgres:
  container_name: exhibition-postgres
  environment:
    POSTGRES_USER: exhibition_user      → user
    POSTGRES_PASSWORD: exhibition_password → pass
    POSTGRES_DB: exhibition_db          → dbname
  ports:
    - "5435:5432"
```

```
profiles.yml 각 값의 출처:

  host   → services: 아래 서비스명 (postgres)
           컨테이너 안 실행 → "postgres" (서비스명)
           로컬 실행        → "localhost"

  port   → 컨테이너 안 실행 → 5432 (내부 포트)
           로컬 실행        → 5435 (매핑 포트)

  user   → POSTGRES_USER 값
  pass   → POSTGRES_PASSWORD 값
  dbname → POSTGRES_DB 값
```

## dbt_project.yml 에서 profile 연결

```yaml
# dbt_exhibition/dbt_project.yml

name: 'dbt_exhibition'
version: '1.0.0'
config-version: 2

profile: 'dbt_exhibition'   # ← profiles.yml 의 최상위 키와 일치해야 함

model-paths: ["models"]

models:
  dbt_exhibition:
    staging:
      +schema: raw_staging
      +materialized: view
    marts:
      +schema: raw_marts
      +materialized: table
```

```
profile 불일치 시:
  Runtime Error: Could not find profile named 'dbt_exhibition'
  → dbt_project.yml 의 profile: 값과
    profiles.yml 의 최상위 키 이름이 정확히 같아야 함
```

## prod 환경 — 언제 필요한가

```
지금 당장은 dev 만으로 충분
아래 상황이 되면 prod 추가:
  팀 협업 → dev 는 개인 / prod 는 공용 DB 분리
  실제 서비스 → 분석가 / BI 도구가 쓰는 DB 따로
  CI/CD 파이프라인 연결
```

```yaml
dbt_exhibition:
  target: dev
  outputs:
    dev:
      type: postgres
      host: postgres
      port: 5432
      user: exhibition_user
      pass: exhibition_password
      dbname: exhibition_db
      schema: public
      threads: 4

    prod:                             # 필요할 때 추가
      type: postgres
      host: prod-server
      port: 5432
      user: prod_user
      pass: "{{ env_var('DBT_PROD_PASSWORD') }}"
      dbname: prod_db
      schema: analytics
      threads: 8
```

```bash
# dev 실행 (기본값)
dbt run

# prod 실행 (명시적으로)
dbt run --target prod
```

---

---

# ④ 연결 테스트

```bash
docker exec -it exhibition-dbt bash
cd /dbt/dbt_exhibition

dbt debug
```

```
정상 출력:
  Connection test: OK connection ok

에러 유형:
  Connection refused    → host 또는 port 잘못됨
  password failed       → pass 확인
  database not found    → dbname 확인
  profile not found     → profile: 이름 불일치
```

---

---

# ⑤ 기본 실행 명령어

```bash
# 컨테이너 접속
docker exec -it exhibition-dbt bash
cd /dbt/dbt_exhibition

# 연결 확인
dbt debug

# 전체 모델 실행
dbt run

# 특정 모델만 실행
dbt run --select stg_exhibitions
dbt run --select mart_current_exhibitions

# 테스트
dbt test

# 문서 생성 + 서버 (localhost:8585)
dbt docs generate
dbt docs serve --host 0.0.0.0 --port 8585
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|이중 폴더 생성|`./dbt_exhibition:/dbt_exhibition` + `dbt init dbt_exhibition`|볼륨을 `./dbt:/dbt` 로 변경|
|`profile not found`|profiles.yml 이름 불일치|`dbt_project.yml` 의 `profile:` 값 확인|
|`Connection refused`|host 잘못됨|컨테이너 안 → `postgres` / 로컬 → `localhost`|
|포트 에러|내부/외부 포트 혼동|컨테이너 안 → `5432` / 로컬 → `5435`|
|`dbt: command not found`|image 방식 사용|`Dockerfile` + `build: .` 방식으로 변경|
|down 후 dbt 사라짐|`image:` 방식 사용|`Dockerfile` + `build: .` 방식으로 변경|
|이미지 변경 미반영|재빌드 안 함|`docker compose up -d --build`|