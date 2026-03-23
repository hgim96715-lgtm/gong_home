---
aliases:
  - Hook
  - 훅
  - 외부연결도구
  - PostgresHook
tags:
  - Airflow
  - Python
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_Operators]]"
  - "[[Airflow_Variables_Connections]]"
  - "[[Python_Database_Connect]]"
  - "[[Airflow_TaskFlow_API]]"
  - "[[SQL_DML_CRUD]]"
---


# Airflow_Hooks — Hook 이란

## 한 줄 요약

```
Airflow Connection UI 에 저장된 정보로
외부 시스템(DB / API / S3 등) 에 연결하는 고수준 인터페이스
Connection ID 만 넘기면 비밀번호 없이 문 따고 들어감
```

---

---

# ① Operator vs Hook — 언제 뭘 쓰나

```
Operator:  완제품 — 정해진 기능만 실행하고 끝
Hook:      부품   — PythonOperator 안에서 커스텀 로직에 사용
```

|항목|Operator|Hook|
|---|---|---|
|사용 방식|DAG 에서 직접 Task 로 사용|PythonOperator 안 함수에서 호출|
|역할|SQL 실행 / API 호출 등 단순 작업|복잡한 로직 / Operator 가 지원 안 하는 기능|
|예시|`PostgresOperator("SELECT ...")`|`hook.copy_expert()` / `hook.get_pandas_df()`|

```python
# Operator — 단순 SQL 실행
task = PostgresOperator(
    task_id="create_table",
    postgres_conn_id="my_postgres",
    sql="CREATE TABLE IF NOT EXISTS ...",
)

# Hook — 커스텀 로직 안에서 사용
def store_data(**context):
    hook = PostgresHook(postgres_conn_id="my_postgres")
    hook.copy_expert(filename="/tmp/data.csv", sql="COPY ...")

task = PythonOperator(task_id="store", python_callable=store_data)
```

---

---

# ② Connection 설정 — 사전 준비

```
Hook 을 쓰려면 Airflow UI 에 연결 정보 등록 필요
Admin → Connections → + 버튼
```

```
PostgreSQL 연결:
  Conn Id:   my_postgres_connection
  Conn Type: Postgres
  Host:      postgres
  Schema:    hospital_db
  Login:     hospital_user
  Password:  hospital_password
  Port:      5432

HTTP 연결:
  Conn Id:   my_http_connection
  Conn Type: HTTP
  Host:      https://swapi.dev
```

---

---

# ③ PostgresHook — 실전 패턴

## 메서드 4가지 비교

|메서드|용도|적합한 상황|
|---|---|---|
|`hook.run(sql)`|SQL 1건 실행|단순 INSERT / UPDATE / DELETE|
|`hook.get_records(sql)`|SELECT → 리스트|소량 조회|
|`hook.get_pandas_df(sql)`|SELECT → DataFrame|pandas 처리 필요|
|`hook.get_conn()` + `execute_values`|대량 INSERT / UPSERT|수백~수천 건 배치|
|`hook.copy_expert(file, sql)`|CSV 파일 통째로 적재|파일 기반 대량 적재|

---

## hook.run() — SQL 1건 실행

```python
hook = PostgresHook(postgres_conn_id="my_postgres_connection")

# 단순 INSERT
hook.run("INSERT INTO logs VALUES (%s)", parameters=("test",))

# UPDATE
hook.run(
    "UPDATE er_realtime SET hvec = %s WHERE hpid = %s",
    parameters=(10, "A1100001")
)
```

```
parameters 는 튜플로 전달
값이 1개여도 ("test",) 처럼 콤마 붙여야 함
단순 1건 실행에 적합 / 대량은 느림
```

---

## hook.get_records() / get_pandas_df() — 조회

```python
# 리스트로 받기
rows = hook.get_records("SELECT hpid, hvec FROM er_realtime LIMIT 5")
# [('A1100001', 10), ('A1100002', 5), ...]

for hpid, hvec in rows:
    print(hpid, hvec)

# DataFrame 으로 받기
df = hook.get_pandas_df("SELECT hpid, hvec FROM er_realtime WHERE hvec > 0")
print(df.head())
```

---

## hook.get_conn() + execute_values — 대량 UPSERT ⭐️

```
Hook 이 Connection ID 로 psycopg2 연결을 만들어줌
그 연결로 직접 커서를 따서 execute_values 사용
→ 수백~수천 건을 한 번의 쿼리로 처리
→ ON CONFLICT (UPSERT) 처리 가능
→ hook.run() 반복보다 압도적으로 빠름
```

```python
from psycopg2.extras import execute_values
from datetime import datetime

def fetch_and_upsert(**context):
    rows = [
        ("A1100001", "서울대병원", "서울시", "02-123", "1",
         37.1234, 127.1234, 500, "Y", "Y", "Y", "Y", "서울"),
        ("A1100002", "세브란스병원", "서울시", "02-456", "1",
         37.5678, 127.5678, 600, "Y", "N", "Y", "N", "서울"),
    ]

    # ① Hook 으로 psycopg2 연결 가져오기
    hook = PostgresHook(postgres_conn_id="postgres_hospital")
    conn = hook.get_conn()
    cur  = conn.cursor()

    # ② UPSERT SQL — VALUES %s (execute_values 전용 문법)
    sql = """
        INSERT INTO er_hospitals (
            hpid, hpname, duty_addr, duty_tel, duty_eryn,
            wgs84_lat, wgs84_lon, hpbdn,
            mk_stroke, mk_cardiac, mk_trauma, mk_pediatric,
            region, updated_at
        ) VALUES %s
        ON CONFLICT (hpid) DO UPDATE SET
            hpname       = EXCLUDED.hpname,
            duty_addr    = EXCLUDED.duty_addr,
            wgs84_lat    = EXCLUDED.wgs84_lat,
            wgs84_lon    = EXCLUDED.wgs84_lon,
            region       = EXCLUDED.region,
            updated_at   = NOW()
    """

    # ③ updated_at 추가 (튜플 이어붙이기)
    # datetime.now() / pendulum.now() / SQL NOW() 중 선택 (아래 비교 참고)
    # [[Python_Lists_Tuples#⑨ 튜플 이어붙이기 — 실전 패턴]] 참고 
    rows_with_ts = [r + (datetime.now(),) for r in rows]

    # ④ execute_values 로 한 방에 적재
    execute_values(cur, sql, rows_with_ts)
    conn.commit()

    cur.close()
    conn.close()
    print(f"[적재] {len(rows)}개 병원 → er_hospitals UPSERT 완료")
```

```
VALUES %s 는 execute_values 전용 문법
  일반 %s 와 다름 — execute_values 가 리스트를 알아서 펼쳐줌

일반 psycopg2:
  sql = "INSERT INTO table (a, b) VALUES (%s, %s)"
  cur.execute(sql, ("값1", "값2"))     ← 한 건씩

execute_values 전용:
  sql = "INSERT INTO table (a, b) VALUES %s"
                                         ↑ 괄호 없이 %s 하나만

  rows = [("A", 1), ("B", 2), ("C", 3)]
  execute_values(cur, sql, rows)
  #              1    2    3
  #              ↑    ↑    ↑
  #            커서  SQL  데이터(리스트)
  → 내부에서 VALUES ('A', 1), ('B', 2), ('C', 3) 으로 변환
  → 쿼리 1번으로 3건 INSERT

  인자 순서:
    1. cur   → psycopg2 커서 (conn.cursor() 로 만든 것)
    2. sql   → VALUES %s 가 포함된 INSERT 문
    3. rows  → [(튜플1), (튜플2), ...] 데이터 리스트

rows_with_ts = [r + (datetime.now(),) for r in rows]:
  각 튜플 r 에 (updated_at,) 를 이어붙임
  r = ("A1100001", ..., "서울")
  r + (datetime.now(),) = ("A1100001", ..., "서울", 2026-03-23 ...)
```

>[[Python_Lists_Tuples#⑨ 튜플 이어붙이기 — 실전 패턴]] 참고 

## updated_at 시각 — datetime.now() vs pendulum.now() vs NOW()

```
3가지 방법이 있고 상황마다 다름
```

```python
from datetime import datetime
import pendulum

# 방법 1: datetime.now() — Python 표준
datetime.now()
# 2026-03-23 12:00:00.123456  ← 로컬 시간 / 타임존 없음 (naive)

# 방법 2: pendulum.now() — Airflow 권장
pendulum.now("Asia/Seoul")
# 2026-03-23 12:00:00.123456+09:00  ← 타임존 포함 (aware)

pendulum.now("UTC")
# 2026-03-23 03:00:00.123456+00:00  ← UTC 기준

# 방법 3: SQL NOW() — DB 서버 시각
# ON CONFLICT DO UPDATE SET updated_at = NOW()
# INSERT 시 DB 서버가 직접 현재 시각 기입
```

```
비교:

  datetime.now()
    타임존 없음 (naive datetime)
    DB 컬럼이 TIMESTAMP (타임존 없음) 면 문제없음
    간단하고 빠름
    단점: 서버 로컬 시각 기준 → 서버 위치 바뀌면 시각 달라짐

  pendulum.now("UTC") 또는 pendulum.now("Asia/Seoul")
    타임존 포함 (aware datetime)
    Airflow 에서 표준으로 사용 (Airflow 자체가 pendulum 사용)
    DB 컬럼이 TIMESTAMPTZ 면 타임존까지 저장
    여러 시간대 서버 운영 시 일관성 보장

  SQL NOW()
    DB 서버 현재 시각
    Python 코드에 시각 포함할 필요 없음
    INSERT 시가 아닌 DB 처리 시각 기준

선택 기준:
  간단한 배치 / 단일 서버  → datetime.now()  (충분)
  Airflow DAG 안에서       → pendulum.now("UTC")  (Airflow 표준)
  UPSERT 갱신 시각만       → SQL NOW()  (가장 간결)
```

## ON CONFLICT + EXCLUDED — UPSERT 핵심 ️

>[[SQL_DML_CRUD#⑤ ON CONFLICT — PostgreSQL UPSERT ⭐️]] 참고 

```
ON CONFLICT (hpid) DO UPDATE SET:
  hpid 가 이미 존재  → INSERT 대신 UPDATE
  hpid 가 없음      → 그냥 INSERT
```

```
EXCLUDED:
  "지금 INSERT 하려다 충돌난 새 데이터" 를 가리키는 가상 테이블
  EXCLUDED.컬럼명 = 지금 넣으려던 새 값
  기존 테이블 값 vs 새로 들어온 값 중 뭘 쓸지 선택할 때 사용
```

```sql
-- 상황: hpid = "A001" 이 이미 DB 에 있을 때
-- 기존: hpname = "구_병원명"
-- 새값: hpname = "신_병원명"

ON CONFLICT (hpid) DO UPDATE SET
    hpname = EXCLUDED.hpname        -- ✅ 새 값으로 덮어씀 → "신_병원명"
    hpname = er_hospitals.hpname    -- 기존 값 유지 → "구_병원명"
```

```
비교:
  EXCLUDED.hpname       새로 들어온 값  (덮어쓸 때)
  er_hospitals.hpname   기존 테이블 값  (유지할 때)

실무 패턴:
  날짜/이름/주소 등 자주 바뀌는 정보 → EXCLUDED (새 값으로 갱신)
  최초 등록일 같은 불변 정보         → 테이블명.컬럼 (기존 유지)
  updated_at                         → NOW() (현재 시각으로 갱신)
```

---

## copy_expert — CSV 파일 통째로 적재

```
파일을 DB 에 통째로 COPY
가장 빠른 방법 / 파일 기반
중간에 Python 가공 필요 없을 때 적합
```

```python
def store_data(**context):
    hook = PostgresHook(postgres_conn_id="my_postgres_connection")
    hook.copy_expert(
        filename="/tmp/data.csv",
        sql="""
            COPY er_hospitals
            FROM stdin
            WITH DELIMITER as ','
        """
    )
```

---

## 세 방법 비교 — 언제 뭘 쓰나

```
hook.run():
  단순 INSERT / UPDATE 1~수십 건
  건별 처리 / 조건 분기 필요할 때

execute_values (get_conn + cursor):
  수백~수천 건 배치 INSERT
  ON CONFLICT UPSERT 처리 필요할 때
  Python 에서 데이터 가공 후 DB 에 넣을 때  ← 이 경우가 가장 많음

copy_expert:
  CSV 파일을 그대로 DB 에 부을 때
  가공 없이 파일 → DB 직통
  가장 빠름 / 파일 형식 맞아야 함
```

|방법|속도|적합한 상황|
|---|---|---|
|`hook.run()`|느림|소량 / 단순 SQL|
|`execute_values`|빠름|대량 배치 / UPSERT|
|`copy_expert`|가장 빠름|CSV 파일 직통 적재|

---

---

# ④ 실전 예제 — API → CSV → DB

## 흐름

```
HttpOperator (API 호출)
    ↓ XCom
PythonOperator (CSV 변환)
    ↓ /tmp/data.csv
PythonOperator + PostgresHook (copy_expert 적재)
```

## DAG 코드

```python
import json
from datetime import datetime
from airflow import DAG
from airflow.providers.http.operators.http import HttpOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from pandas import json_normalize


def _transform(**context):
    """API 응답 → CSV 파일로 변환"""
    ti = context["ti"]
    response = ti.xcom_pull(task_ids="get_op")

    df = json_normalize({
        "name":   response["name"],
        "height": response["height"],
        "mass":   response["mass"],
    })
    df.to_csv("/tmp/swapi.csv", header=False, index=False)


def _load(**context):
    """CSV → PostgreSQL COPY 적재"""
    hook = PostgresHook(postgres_conn_id="my_postgres_connection")
    hook.copy_expert(
        filename="/tmp/swapi.csv",
        sql="""
            COPY swapi_characters
            FROM stdin
            WITH DELIMITER as ','
        """
    )


with DAG(
    dag_id="api_to_postgres",
    schedule="0 0 * * *",
    start_date=datetime(2026, 1, 1),
    catchup=False,
) as dag:

    create_table = SQLExecuteQueryOperator(
        task_id="create_table",
        conn_id="my_postgres_connection",
        sql="""
            CREATE TABLE IF NOT EXISTS swapi_characters (
                name   TEXT NOT NULL,
                height TEXT,
                mass   TEXT
            );
        """
    )

    get_data = HttpOperator(
        task_id="get_op",
        http_conn_id="my_http_connection",
        endpoint="/api/people/1/",
        method="GET",
        response_filter=lambda r: json.loads(r.text),
    )

    transform = PythonOperator(
        task_id="transform",
        python_callable=_transform,
    )

    load = PythonOperator(
        task_id="load",
        python_callable=_load,
    )

    create_table >> get_data >> transform >> load
```

---

---

# ⑤ 주의사항

## Hook 은 반드시 함수 안에서 생성

```python
# ❌ Top-level 에서 Hook 생성 — 절대 금지
hook = PostgresHook(postgres_conn_id="my_postgres")  # DAG 파일 최상단

# ✅ 함수 안에서 생성
def my_task(**context):
    hook = PostgresHook(postgres_conn_id="my_postgres")  # 실행 시점에 생성
```

```
DAG 파일 최상단에서 Hook 생성하면:
  Airflow Scheduler 가 DAG 파일을 파싱할 때마다 (30초마다)
  DB 접속 시도 → 연결 수 폭발 → 서버 다운 위험
```

## Provider 패키지 설치 필요

```bash
# PostgresHook
pip install apache-airflow-providers-postgres

# HttpHook
pip install apache-airflow-providers-http

# S3Hook
pip install apache-airflow-providers-amazon
```

---

---

# 한눈에 정리

```
Hook:
  Connection ID 만으로 외부 시스템 연결
  PythonOperator 안 함수에서 사용
  Operator 가 지원 안 하는 기능 사용 가능

PostgresHook 주요 메서드:
  hook.run(sql, parameters)    → SQL 1건 실행
  hook.get_records(sql)        → 결과 리스트
  hook.get_pandas_df(sql)      → DataFrame
  hook.get_conn()              → psycopg2 연결 (execute_values 용)
  hook.copy_expert(file, sql)  → CSV 파일 통째로 COPY

대량 처리 선택 기준:
  Python 에서 가공 후 DB 저장  → get_conn() + execute_values
  CSV 파일을 DB 에 직통        → copy_expert
  소량 / 단순 SQL             → hook.run()

주의:
  함수 안에서 생성 (Top-level ❌)
  Provider 패키지 별도 설치 필요
  execute_values 는 VALUES %s (일반 %s 아님)
```