---
aliases:
  - Connections
  - Variables
  - BaseHook
  - Secrets
  - 보안 관리
  - DB연결
  - deserialize_json
  - default_var
tags:
  - Airflow
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_Hooks]]"
  - "[[Python_Database_Connect]]"
---
# Airflow_Variables_Connectionss — Airflow 금고

## 한 줄 요약

```
Connections  DB 접속 정보 저장 (host / user / password 묶음)
Variables    공통 설정값 저장 (API Key / 경로 / 환경 설정)

코드에 비밀번호 직접 쓰지 않고
Airflow UI 금고에 저장 → Connection ID / Key 로 호출
```

---

---

# ① Connections vs Variables

|항목|Connections|Variables|
|---|---|---|
|위치|Admin → Connections|Admin → Variables|
|용도|접속 정보 (DB / API / SSH)|설정값 (경로 / 버전 / 상수)|
|코드 호출|`BaseHook.get_connection(ID)`|`Variable.get(key)`|
|반환값|Connection 객체|문자열 또는 딕셔너리|

```
"로그인해야 들어갈 수 있는 곳" → Connections
"그냥 값만 필요한 것"          → Variables
```

---

---

# ② Connections — 접속 정보 관리

## UI 등록

```
Admin → Connections → + 버튼

필드:
  Connection Id:   my_postgres_connection  ← 코드에서 부를 이름
  Connection Type: Postgres
  Host:            postgres
  Schema:          hospital_db
  Login:           hospital_user
  Password:        hospital_password
  Port:            5432
```

```
Connection Id 규칙:
  영어 소문자 + 언더바 조합 (snake_case)
  한글 / 특수문자 비추천 (인코딩 문제)
```

## BaseHook 으로 꺼내기

```python
from airflow.hooks.base import BaseHook

def my_task(**context):
    # Connection ID 로 객체 가져오기
    conn = BaseHook.get_connection("my_postgres_connection")

    # 점(.) 으로 속성 접근
    print(conn.host)       # postgres
    print(conn.login)      # hospital_user
    print(conn.password)   # hospital_password  ← 로그에서 자동 마스킹됨
    print(conn.port)       # 5432
    print(conn.schema)     # hospital_db
```

## conn 속성 정리

|코드|UI 입력창|예시|
|---|---|---|
|`conn.host`|Host|`postgres`|
|`conn.login`|Login|`hospital_user`|
|`conn.password`|Password|`hospital_password`|
|`conn.port`|Port|`5432`|
|`conn.schema`|Schema|`hospital_db`|
|`conn.extra`|Extra|`{"ssl": true}`|

```
⚠️ conn 자체를 print 하면 객체 껍데기만 나옴
   conn.host 처럼 점(.) 으로 꺼내야 값이 나옴

⚠️ 로그에 비밀번호 찍으면 안 됨 (Airflow 가 자동 마스킹하긴 하지만 습관)
```

## BaseHook vs 전용 Hook

```
BaseHook.get_connection():
  정보만 가져올 때 (어떤 DB 든 상관없이)
  연결 정보만 필요하고 직접 psycopg2 등 연결할 때

PostgresHook:
  Postgres 전용 → SQL 실행 / 데이터 조회까지 가능
  hook.run() / hook.get_pandas_df() / hook.copy_expert()
  → [[Airflow_Hooks]] 참고
```

---

---

# ③ Variables — 설정값 관리

## UI 등록

```
Admin → Variables → + 버튼

단순 값:
  Key:  api_key
  Val:  abc123xyz

JSON 묶음:
  Key:  db_config
  Val:  {"host": "postgres", "port": "5432", "db": "hospital_db"}
```

## Variable.get() 으로 꺼내기

```python
from airflow.models import Variable

def my_task(**context):

    # 단순 값
    api_key = Variable.get("api_key")
    print(api_key)   # "abc123xyz"

    # JSON → 딕셔너리로 자동 변환
    config = Variable.get("db_config", deserialize_json=True)
    print(config["host"])   # "postgres"
    print(config["port"])   # "5432"

    # 키가 없을 때 기본값 (없으면 DAG 에러)
    value = Variable.get("optional_key", default_var="기본값")
```

## Variable.get() 파라미터

```python
Variable.get(key, deserialize_json=False, default_var=None)
```

|파라미터|설명|기본값|
|---|---|---|
|`key`|Variables 에 등록한 Key 이름|필수|
|`deserialize_json`|`True` → dict 자동 변환 / `False` → 문자열|`False`|
|`default_var`|키 없으면 에러 대신 이 값 반환|`None`|

```python
# deserialize_json 비교
# UI 에 {"a": 1} 저장했을 때

Variable.get("key")                       # '{"a": 1}'  ← 문자열
Variable.get("key", deserialize_json=True) # {'a': 1}   ← 딕셔너리 ✅
```

```
실무 팁:
  관련 설정 여러 개를 JSON 하나로 묶어서 등록
  → Variables 수 줄이고 관리 편해짐
  deserialize_json=True 와 함께 사용

  default_var 습관적으로 설정
  → 누가 UI 에서 변수 지워도 DAG 에러 방지
```

---

---

# ④ 트러블슈팅 — env_file 용량 에러

## 증상

```yaml
# docker-compose.yml
x-airflow-common: &airflow-common
  env_file:
    - .env   # ← 추가 후 에러
```

```
docker.errors.APIError: request body too large
```

## 원인

```
.env 파일 내용이 Docker API 요청 허용 크기 초과
.env 에 긴 API 키 / 긴 경로 등이 있을수록 발생
```

## 해결 — environment + Variable.get 조합

```python
# DAG 파일에서
from airflow.models import Variable

run_task = DockerOperator(
    task_id="run",
    environment={
        "DB_HOST":     Variable.get("db_host"),      # Airflow Variables 에서 꺼냄
        "DB_PASSWORD": Variable.get("db_password"),
        "DB_PORT":     "5432",
        "DB_NAME":     "hospital_db",
    }
)
```

```python
# 컨테이너 안 스크립트에서
import os
db_host = os.getenv("DB_HOST")   # ✅ environment 로 주입됐으니 정상 읽힘
```

```
흐름:
  Airflow Variables → Variable.get() → environment={} 에 담김
  → DockerOperator 가 컨테이너 실행 시 환경변수로 주입
  → 컨테이너 안 os.getenv() 로 읽기

주입 → 읽기 순서가 핵심
os.getenv() 단독으로는 안 됨 (주입 없이 읽기 불가)
```

|방법|결과|이유|
|---|---|---|
|`env_file=".env"`|❌ 용량 에러|Docker API 크기 제한|
|`os.getenv()` 단독|❌ None|컨테이너에 주입 안 됨|
|`environment` + `Variable.get()`|✅|Variables → DAG → 컨테이너 순서|

---

---
