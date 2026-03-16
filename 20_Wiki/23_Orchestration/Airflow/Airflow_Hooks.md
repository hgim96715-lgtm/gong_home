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
  - "[[Airflow_Providers]]"
  - "[[]]"
  - "[[Variables_Connections]]"
  - "[[Airflow_Operators]]"
  - "[[Pandas_Json_Normalize]]"
  - "[[Python_JSON]]"
  - "[[Python_Requests_Response]]"
  - "[[Variables_Connections]]"
---
## 개념 한 줄 요약

Hook은 외부 플랫폼(AWS, S3, Postgres, Slack 등)과 **상호작용하기 위한 고수준 인터페이스(High-level Interface)** 입니다.
쉽게 말해, 복잡한 저수준 API 코드 없이도 **"Connection ID만 던져주면 알아서 문 따고 들어가는 만능 열쇠"** 입니다.

---
## 왜 필요한가 (Why)

**문제점:**
- 파이썬 코드 안에서 DB에 접속하려면 `import psycopg2`, `connect(host=..., user=...)` 처럼 복잡한 연결 코드를 매번 짜야 합니다.
- 비밀번호가 코드에 노출될 위험이 있습니다.
- **`PostgresOperator`는 SQL 실행만 가능**할 뿐, CSV 파일을 통째로 DB에 밀어넣는(`COPY`) 등의 복잡한 작업은 불가능합니다.

**해결책:**
- Hook은 **Airflow Connection UI에 저장된 정보**를 자동으로 읽어옵니다.
- 개발자는 "Postgres에 연결해줘"라고만 하면, Hook이 알아서 인증하고 커서(Cursor)를 가져다줍니다.
- **`copy_expert`** 같은 고급 메소드를 제공하여 데이터 엔지니어링의 핵심인 **대량 데이터 처리**를 가능하게 합니다.

---
## Practical Context (Operator vs Hook)

초보자가 가장 헷갈리는 부분입니다.

* **Operator (`PostgresOperator`):**
    * "그냥 SQL 한 줄 실행하고 끝내줘."
    * **완제품**입니다. 정해진 기능(SQL 실행)만 수행하고 퇴근합니다.

* **Hook (`PostgresHook`):**
    * "SQL로 데이터를 가져와서, 파이썬으로 가공한 다음, 다시 다른 API로 쏘고 싶어."
    * **부품(도구)** 입니다. 주로 **`PythonOperator` 안에서** 내 마음대로 커스텀 로직을 짤 때 불러와서 씁니다.

---
## Code Core Points (실전 예제)

**상황:** 스타워즈 API(SWAPI)에서 데이터를 가져와(Http), Postgres DB에 저장하는 파이프라인.
**시나리오:** HTTP API로 데이터를 가져와 CSV로 만든 뒤, **Hook의 `copy_expert` 기능을 이용해 DB에 고속으로 적재**합니다.

>https://swapi.info/

### 1. Connection 설정 (준비물)

Hook을 쓰려면 웹 UI에서 연결 정보를 먼저 만들어야 합니다.

- **HTTP 연결:** `my_http_connection` (Host: `https://swapi.dev`)
- **DB 연결:** `my_postgres_connection` (Login/PW: `airflow`)

### 2. PythonOperator 내부에서 Hook 사용하기

```python
import json
from datetime import datetime
from airflow import DAG
from airflow.providers.http.operators.http import HttpOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from pandas import json_normalize

# 1. API 결과를 받아서 CSV로 변환하는 함수
def _extract_date(**context):
    ti = context["ti"]
    # XCom을 통해 앞선 태스크(get_op)가 받은 데이터를 꺼내옵니다.
    response = ti.xcom_pull(task_ids="get_op")
	# JSON 데이터를 Pandas DataFrame으로 예쁘게 폅니다.
    starwars_character = json_normalize({
        "name": response["name"],
        "height": response["height"],
        "mass": response["mass"]
    })

   # DB에 넣기 좋게 CSV 파일로 임시 저장합니다 (헤더/인덱스 제거).
    starwars_character.to_csv(
        "/tmp/starwars_character.csv",
        header=False,
        index=False
    )

# 2. CSV → PostgreSQL COPY  ( Hook을 사용해 CSV를 DB에 '부어버리는' 함수)
def _store_character(**context):
	# Connection ID만 있으면 비밀번호 없이 바로 연결 객체 생성!
    hook = PostgresHook(postgres_conn_id="my_postgres_connection")
	# [핵심] SQL 한 줄씩 실행하는 게 아니라, 파일을 통째로 복사(COPY)합니다.
	# 이건 Operator로는 못 하고 Hook이라서 가능한 기능입니다.
    hook.copy_expert(
        filename="/tmp/starwars_character.csv",
        sql="""
            COPY starwars_character
            FROM stdin
            WITH DELIMITER as ','
        """
    )

with DAG(
    dag_id="http_dag",
    description="http dag",
    schedule="0 0 * * *",
    start_date=datetime(2026, 1, 10),
    catchup=False,
) as dag:

    # 테이블 만들기 (Operator 사용)
    task_create_table_op = SQLExecuteQueryOperator(
        task_id="create_table_op",
        conn_id="my_postgres_connection",
        sql="""
            CREATE TABLE IF NOT EXISTS starwars_character (
                name TEXT NOT NULL,
                height TEXT NOT NULL,
                mass TEXT NOT NULL
            );
        """
    )

    # API 호출하기 (Operator 사용)
    task_get_op = HttpOperator(
        task_id="get_op",
        http_conn_id="my_http_connection",
        endpoint="/api/people/1/",
        method="GET",
        headers={"Content-Type": "application/json"},
        response_filter=lambda response: json.loads(response.text),
        log_response=True,
    )

    ## 데이터 가공 (Python 함수 실행)
    task_extract_data_op = PythonOperator(
        task_id="extract_data_op",
        python_callable=_extract_date,
    )

    # 데이터 적재 (Hook을 사용하는 Python 함수 실행)
    task_store_op = PythonOperator(
        task_id="store_op",
        python_callable=_store_character,
    )

    task_create_table_op >> task_get_op >> task_extract_data_op >> task_store_op

```

##  Detailed Analysis (구조 이해)

**`SQLExecuteQueryOperator` vs `PostgresHook`**:

- 테이블을 만드는 단순 SQL(`CREATE TABLE`)은 편하게 **Operator**를 썼습니다.
- 하지만 데이터를 넣을 때는 성능을 위해 `INSERT INTO` 대신 `COPY` 명령어를 써야 했고, 이를 지원하는 **Hook의 `copy_expert`** 를 사용하기 위해 **PythonOperator** 안에서 Hook을 불러왔습니다.

**데이터 흐름 (Data Flow)**:

- `HttpOperator` (데이터 수집) -> **XCom** (메모리 전달) -> `_extract_date` (CSV 변환) -> **로컬 파일** (`/tmp/...`) -> `_store_character` (Hook으로 적재)

---
## Common Beginner Misconceptions

1. **"Hook은 DAG 파일 최상단에서 실행하면 되나요?"**
    - **절대 안 됩니다!** Hook은 반드시 `def 함수명():` 안에서(실행 시점) 생성해야 합니다. 
    - DAG 파일 최상단(Top-level)에서 Hook으로 DB에 연결하면, Airflow 스케줄러가 DAG 파일을 파싱할 때마다 DB 접속을 시도하여 **서버를 다운**시킬 수 있습니다.

2. **"Provider 설치가 필요한가요?"**
    - 네! `PostgresHook`을 쓰려면 `apache-airflow-providers-postgres` 패키지가 설치되어 있어야 합니다.

3. **"Hook은 언제 쓰나요?"**:
	- Operator가 제공하지 않는 **특수 기능** (예: `copy_expert`, `get_pandas_df`)이 필요할 때 씁니다.
	- 복잡한 **Python 로직 중간에 DB나 외부 API 호출**이 필요할 때 씁니다.