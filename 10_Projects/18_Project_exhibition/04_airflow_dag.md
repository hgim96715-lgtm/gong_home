---
aliases:
  - airflow
tags:
  - Project
related:
  - "[[00_exhibition_Project]]"
  - "[[Airflow_Hooks]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Airflow_DAG_Concept]]"
  - "[[Airflow_Operators]]"
---
# 05 Airflow DAG — Exhibition Pipeline

## 파이프라인 흐름

```
crawl → load → dbt_run → summary
```

|Task|내용|
|---|---|
|`crawl`|InterparkCrawler.crawl_all() → return 으로 XCom 자동 저장|
|`load`|crawl 결과 인자로 받아 PostgreSQL 5개 함수 순서대로 적재|
|`dbt_run`|subprocess 로 dbt run 실행 (staging → marts)|
|`summary`|적재 건수 로그 출력|

스케줄: **매일 KST 06:00** (pendulum tz 설정)

---

## 파일 위치

```
airflow/
└── dags/
    └── exhibition_pipeline_dag.py
```

docker-compose.yml 마운트:

```yaml
volumes:
  - ./airflow/dags:/opt/airflow/dags
  - ./crawl:/opt/airflow/crawl
  - ./crawl/load:/opt/airflow/load
  - ./dbt:/opt/airflow/dbt
```

---

## 초기 설정 (최초 1회)

```bash
# 1. 컨테이너 실행 (Airflow DB 초기화 + admin 계정은 커맨드에 포함)
docker compose up -d

# 2. 접속
# http://localhost:8085
# ID: admin / PW: admin
```

---

## DAG 실행 방법

### UI 에서 수동 실행

```
DAGs → exhibition_pipeline → Trigger DAG (▶ 버튼)
```

### CLI 로 수동 실행

```bash
docker exec -it exhibition-airflow \
  airflow dags trigger exhibition_pipeline
```

### 특정 태스크만 테스트

```bash
# crawl 단독
docker exec -it exhibition-airflow \
  airflow tasks test exhibition_pipeline crawl 2026-04-30

# load 단독
docker exec -it exhibition-airflow \
  airflow tasks test exhibition_pipeline load 2026-04-30

# dbt_run 단독
docker exec -it exhibition-airflow \
  airflow tasks test exhibition_pipeline dbt_run 2026-04-30
```

---

## 핵심 개념

### @dag / @task 데코레이터 (TaskFlow API)

Airflow 2.0 이후 권장 방식.  
`PythonOperator` 를 직접 쓰는 것보다 코드가 훨씬 깔끔하다.

```python
# PythonOperator 방식 (구식)
t_crawl = PythonOperator(task_id="crawl", python_callable=task_crawl, dag=dag)

# @task 데코레이터 방식 (현재)
@task
def crawl() -> dict:
    ...
    return {...}   # return 값이 자동으로 XCom 에 저장됨
```

### XCom — 태스크 간 데이터 전달

`@task` 에서는 `return` 값이 자동으로 XCom 에 저장되고,  
다음 태스크 함수의 인자로 자동으로 전달된다.

```python
# return 으로 저장 (xcom_push 불필요)
@task
def crawl() -> dict:
    return {"exhibitions": [...], "price_rows": [...], "stats": [...]}

# 함수 인자로 수신 (xcom_pull 불필요)
@task
def load(crawl_result: dict) -> dict:
    exhibitions = crawl_result["exhibitions"]
    ...
```

> ⚠️ XCom 은 작은 데이터 전달용 (Airflow 메타DB 저장).  
> 전시 50건 규모에서는 충분하지만, 대용량이면 S3/GCS 등 외부 저장소 활용 권장.

### 의존 관계 정의

```python
crawl_result = crawl()
load_result  = load(crawl_result)   # crawl_result 를 인자로 받으면 자동으로 의존 관계 설정
dbt_task     = dbt_run()
summary_task = summary(load_result)

# dbt 는 load 뒤에 실행, summary 는 dbt 뒤에
load_result >> dbt_task >> summary_task
```

`crawl → load` 는 함수 인자 연결로 자동 설정.  
`load → dbt_run → summary` 는 `>>` 연산자로 명시.

### schedule_interval + pendulum timezone

```python
start_date=pendulum.datetime(2026, 4, 30, tz="Asia/Seoul"),
schedule_interval="0 6 * * *",
# pendulum 으로 tz 지정하면 cron 도 그 타임존 기준으로 해석됨
# → 매일 KST 06:00 실행
```

---

---

## Scheduler / Webserver 분리 구조

### 문제 — 단일 컨테이너 & 방식

```bash
# ❌ 기존: 하나의 컨테이너에서 & 로 백그라운드 실행
airflow scheduler &   # 죽어도 모름, 재시작 없음
airflow webserver     # 포그라운드만 살아있음
# → DAG 상태가 None 으로 보임 (scheduler 가 실제로 안 돌고 있음)
```

### 해결 — 서비스 3개로 분리

```yaml
airflow-init:       # 최초 1회 DB 초기화 + 계정 생성 후 종료 (restart: no)
airflow-scheduler:  # DAG 감지 + 실행 담당 (restart: always)
airflow-webserver:  # UI 담당 (restart: always)
```

`restart: always` 로 scheduler 가 죽으면 자동 재시작.  
scheduler / webserver 가 `airflow_logs` 볼륨을 공유해서 UI 에서 로그 확인 가능.

### 컨테이너 이름 변경

|기존|변경 후|
|---|---|
|`exhibition-airflow`|`exhibition-airflow-scheduler`|
|(없음)|`exhibition-airflow-webserver`|
|(없음)|`exhibition-airflow-init`|

CLI 명령어도 컨테이너 이름 변경:

```bash
# DAG 트리거
docker exec -it exhibition-airflow-scheduler \
  airflow dags trigger exhibition_pipeline

# 태스크 테스트
docker exec -it exhibition-airflow-scheduler \
  airflow tasks test exhibition_pipeline crawl 2026-04-30

# 로그 확인
docker exec -it exhibition-airflow-scheduler \
  airflow tasks logs exhibition_pipeline load 2026-04-30

# 스케줄러 상태 확인
docker exec -it exhibition-airflow-scheduler \
  airflow jobs check --job-type SchedulerJob
```

### 기동 순서

```bash
# 1. 전체 실행
docker compose up -d

# 2. init 완료 확인 (DB 초기화 + 계정 생성 후 자동 종료)
docker compose logs airflow-init

# 3. scheduler / webserver 정상 기동 확인
docker compose logs -f airflow-scheduler
docker compose logs -f airflow-webserver

# 4. UI 접속
# http://localhost:8085  (ID: admin / PW: admin)
```

> ⚠️ `airflow-init` 이 먼저 완료돼야 scheduler / webserver 가 정상 동작.  
> init 이 아직 실행 중이면 잠시 기다렸다가 UI 접속할 것.

---

## 실패 처리

```python
default_args = {
    "retries":     1,
    "retry_delay": timedelta(minutes=5),
}
```

각 태스크에서 명시적 예외 발생:

```python
if not exhibitions:
    raise ValueError("수집된 전시가 없습니다.")   # crawl 실패

if not loader.test_connection():
    raise ConnectionError("DB 연결 실패")          # load 실패

if result.returncode != 0:
    raise RuntimeError(f"dbt run 실패")            # dbt 실패
```

---

## 로그 확인

```bash
# 태스크 로그 (UI 에서도 확인 가능)
docker exec -it exhibition-airflow \
  airflow tasks logs exhibition_pipeline load 2026-04-30

# 컨테이너 전체 로그
docker compose logs -f airflow
```

---

## 실제 실행 에러 기록

### upsert_exhibition_prices — ON CONFLICT 중복 에러

```
psycopg2.errors.CardinalityViolation:
ON CONFLICT DO UPDATE command cannot affect row a second time
```

**원인:** `price_type_code` 가 bestprices API 에서 `None` 으로 오고,  
prices/group API 에서 `""` (빈 문자열) 로 와서  
Python dict 에서는 다른 키지만 DB UNIQUE 조건에서는 같은 행으로 충돌.

**해결:** dedup key 에서 `or ""` 로 통일.

```python
# 수정 전
key = (eid, p.get("seat_grade"), p.get("price_grade"), p.get("price_type_code"))
# → None 과 "" 가 다른 키 → dedup 통과 → DB 충돌

# 수정 후
key = (
    eid,
    p.get("seat_grade")      or "",
    p.get("price_grade")     or "",
    p.get("price_type_code") or "",   # None → "" 로 통일
)
```

---

## PENDING

- dbt test 태스크 추가 (dbt_run 뒤에)
- 크롤링 실패 시 Slack 알림 추가
- Superset 데이터셋 캐시 갱신 태스크 추가