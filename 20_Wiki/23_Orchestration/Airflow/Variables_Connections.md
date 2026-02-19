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
  - Security
related:
  - "[[Airflow_Hooks]]"
  - "[[Airflow_UI_Usage]]"
  - "[[00_Airflow_HomePage]]"
---
## 개념 한 줄 요약

**Connections & Variables**는 Airflow의 **"공용 금고"** 야.
* **Connections:** DB 주소, ID, Password 같은 **민감한 접속 정보**를 저장. (보안 중요 🔒)
* **Variables:** 파이프라인에서 공통으로 쓰는 **상수 값**(예: 환경 설정, 모델 버전)을 저장. (공유 중요 📢)

---
## 왜 필요한가 (Why)

**문제점 (Hardcoding):**
- 코드 안에 `password="1234"`라고 적어서 깃허브에 올리면? **해킹당함.** 

**해결책:**
- 코드는 `get_connection('my_db')`라고만 적고, 진짜 비밀번호는 **Airflow 웹 UI (Admin -> Connections)** 에 따로 저장해두는 거야.

---
## BaseHook vs Variable.get() — 언제 뭘 써야 하나?

|            | `BaseHook.get_connection()`              | `Variable.get()`  |
| ---------- | ---------------------------------------- | ----------------- |
| **저장 위치**  | Admin → Connections                      | Admin → Variables |
| **용도**     | DB 접속 정보 (host, port, user, password 묶음) | 단순 설정값, 개별 값      |
| **꺼내는 방식** | `conn.host`, `conn.password` 속성으로 접근     | 바로 값으로 반환         |


---
##  Practical Context (UI 설정법)

코드를 돌리기 전에 먼저 UI에 정보를 등록해야 해. 

1.  **관리자 메뉴 진입:** 왼쪽 사이드바에서 **[관리자 (Admin, 톱니바퀴 아이콘)]** 클릭.
2.  **커넥션 선택:** 메뉴 리스트에서 **[커넥션 (Connections)]** 선택.
3.  **생성:** 파란색 **(+)** 버튼 클릭.
4.  **정보 입력:**
	- **Connection Id:** `my_postgres_connection` (코드에서 부를 이름)!! 여기서 만듬 ! 
	- **Connection Type:** `Postgres` (혹은 Generic)
	- **Host, Login, Password:** 실제 접속 정보 입력.

---
##  Code Core Points (꺼내 쓰는 법)

네가 작성한 코드는 **`BaseHook`** 을 사용해서 금고 문을 따는 열쇠 같은 코드야.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
# ⭐️ 핵심 모듈: 훅(Hook)들의 대장님
from airflow.hooks.base import BaseHook
import pendulum

def _access_connection(ti):
    print("🔐 Connection 정보를 가져옵니다...")
    
    # 1. 금고에서 ID로 정보 꺼내오기 (객체 리턴)
    conn = BaseHook.get_connection('my_postgres_connection')
    
    # 2. 필요한 정보 속성(Attribute)으로 접근
    print(f"Host: {conn.host}")
    print(f"Login: {conn.login}")
    
    # 🚨 주의: 비밀번호는 실제 운영 로그에는 절대 찍으면 안 돼! (실습용이라 찍음)
    print(f"Password: {conn.password}")
    print("ended")

with DAG(
    dag_id='connection_access',
    start_date=pendulum.datetime(2026, 1, 20, tz="Asia/Seoul"),
    catchup=False
) as dag:
    
    task_extract_data = PythonOperator(
        task_id="extract_data_op",
        python_callable=_access_connection
    )
```

---
### 💡 Deep Dive: `get_connection`의 정체

- `{python}BaseHook.get_connection(불러올 connection의 ID)`

```python
# 1. 명령 내리기 (Fetch)
# BaseHook.get_connection(불러올 connection 테이블)
conn = BaseHook.get_connection('my_postgres_connection')

# SQL로 생각하기
SELECT * FROM connection
WHERE conn_id = 'my_postgres_connection'; # 👈 함수에 넣어준 그 문자열!
```

이 코드가 실행되면 Airflow는 메타데이터 DB(`connection` 테이블)에 가서 해당 ID를 찾습니다. 
그리고 그 결과를 **`Connection` 객체(Object)** 라는 **"종합 선물 세트"** 로 포장해서 `conn` 변수에 담아줍니다.

#### 선물 세트(`conn`) 언박싱 하기

`conn` 안에는 우리가 UI에 입력했던 정보들이 **점(.) 속성**으로 들어있어.

|**코드 (conn.속성)**|**UI 입력창 매칭**|**실제 값 예시**|
|---|---|---|
|`conn.host`|Host|`127.0.0.1`|
|`conn.login`|Login|`my_user`|
|`conn.password`|Password|`secret1234`|
|`conn.port`|Port|`5432`|
|`conn.schema`|Schema|`my_db`|
|`conn.extra`|Extra|`{"ssl": true}` (JSON형식)|

**중요:**
- `conn` 자체를 출력(`print(conn)`)하면 그냥 "객체 껍데기"만 보일 수 있어. 
- 반드시 `conn.host` 처럼 **점(.)을 찍어서 내용물을 꺼내야 해!**

---
## Detailed Analysis (Hooks vs BaseHook)

**BaseHook:**
- "어떤 자물쇠든 다 열 수 있는 마스터키.
- DB 종류 상관없이 정보를 가져올 때 씀.

**PostgresHook / MySqlHook:**
- 특정 자물쇠 전용 키
- 정보를 가져오는 것뿐만 아니라, **SQL을 실행(`run`)하거나 데이터를 뽑는(`get_pandas_df`)** 기능까지 포함되어 있음.
- 단순히 정보만 필요하면 `BaseHook`도 OK, 쿼리를 날릴 거면 전용 Hook을 쓰는 게 좋아.

---
## 초보자가 자주 착각하는 포인트

1. "로그에 비밀번호가 안 보여요!"
	- Airflow는 똑똑해서 로그에 `password` 같은 단어가 들어가면 자동으로 `***`로 가려버려(Masking).
	-  안 보인다고 당황하지 마. 실제 변수에는 값이 잘 들어와 있어.

2. "Connection ID를 한글로 해도 되나요?"
	- 비추천! 코드에서 불러올 때 인코딩 문제가 생길 수 있으니 **영어 소문자 + 언더바(`_`)** 조합(Snake Case)을 쓰는 게 국룰이야.

>tip 
>면접관이 **"Airflow에서 DB 비밀번호 어떻게 관리했어요?"** 라고 물으면 이렇게 대답하세요.
>"하드코딩하지 않고 **Airflow Connection** 기능을 사용해 메타데이터 DB에 암호화하여 저장했습니다. 
>코드에서는 `BaseHook`이나 특정 Hook을 통해 **Connection ID**로만 호출하여 보안성을 높였습니다."

---
## Variables (변수): 

설정값의 저장소 **Connection**이 "비밀번호 금고"라면, **Variable**은 "공용 게시판"이야. 
API Key, 파일 경로, 모델 버전, 환경 설정(Dev/Prod) 같은 **상수 값(Key-Value)** 을 저장할 때 써.

### A. Connection vs Variable (절대 헷갈리지 말기!)

초보자가 가장 많이 하는 실수! **"변수를 Connection 메뉴에 적고 못 찾겠대."**

| 구분        | **Connection (커넥션)**           | **Variable (변수)**          |
| :-------- | :----------------------------- | :------------------------- |
| **메뉴 위치** | Admin -> **Connections**       | Admin -> **Variables**     |
| **용도**    | **접속 정보** (DB, AWS, SSH)       | **설정 값** (경로, 버전, JSON 설정) |
| **필수 항목** | Host, Schema, Login, Password  | Key, Val                   |
| **코드 호출** | `BaseHook.get_connection(...)` | `Variable.get(...)`        |

> **💡 핵심:** "로그인해야 들어갈 수 있는 곳"은 **Connection**, "그냥 값만 필요한 것"은 **Variable**이야!

### B. Practical Context (JSON 딕셔너리 활용법)

변수가 100개라고 100개를 다 등록하면 관리하기 힘들어. 
그래서 실무에서는 **관련된 설정들을 JSON(딕셔너리) 하나로 묶어서** 저장해.

1. **UI 등록:** Admin -> Variables -> **(+) 변수 추가**
2. **Key:** `var_dict`
3. **Val:** `{"key1": "hello", "key2": "world"}` (JSON 형식으로 입력)

![[스크린샷 2026-01-23 오후 5.08.40.png|670x240]]

### C. Code Core Points (JSON 꺼내기)

이  코드는 JSON 형태의 텍스트를 **자동으로 파이썬 딕셔너리로 변환(`deserialize`)** 해서 가져오는 고급 기술이야.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable
import pendulum


def _access_variable(ti):
    print("Fetching variable info...")

    # Variable을 JSON → dict로 가져오기
    # 핵심: deserialize_json=True
    # 이걸 안 쓰면 그냥 문자열 '{"key1": ...}'로 가져와져서 딕셔너리 처럼 못 써.
    #  True로 하면 파이썬 dict {key1: val1} 로 자동 변환됨!
    var_dict = Variable.get("var_dict", deserialize_json=True)

    print(var_dict["key1"])  # hello
    print(var_dict["key2"])  # world
    print("ended")


with DAG(
    dag_id="access_variable",
    start_date=pendulum.datetime(2026, 1, 20, tz="Asia/Seoul"),
    catchup=False,
) as dag:
    task_extract_data_op = PythonOperator(
        task_id="extract_data_op",
        python_callable=_access_variable,
    )

```

### D. 문법 상세: `Variable.get()` 뜯어보기

함수 괄호 안에 뭘 넣어야 하는지 헷갈린다면 이것만 기억해!

```python
Variable.get(key, deserialize_json=False, default_var=None)
```

| **인자 (Argument)**      | **필수** | **설명**                                                                                                                                     |
| ---------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **`key`**              | ✅      | **변수 이름.** UI에서 등록한 `Key` 값을 문자열로 적어야 해. (예: `"var_dict"`)                                                                                 |
| **`deserialize_json`** | ❌      | **JSON 변환 스위치.** (기본값: `False`)<br><br>- `True`: 텍스트를 **딕셔너리(dict)나 리스트(list)** 로 자동 변환해 줘.<br><br>- `False`: 그냥 **문자열(String)** 덩어리로 가져와. |
| **`default_var`**      | ❌      | **안전장치(Default Value).**  <br><br>- 만약 UI에 해당 Key가 없으면 에러가 나면서 DAG가 터지는데, 이 값을 적어두면 에러 대신 **이 기본값을 반환**해.                                  |


#### 💡 `deserialize_json` 옵션 비교 (Before & After)

**UI에 `{"a": 1}` 이라고 저장되어 있을 때:**
1. `deserialize_json=False` (기본)
	- 결과: `'{"a": 1}'` (그냥 글자)
	- 문제: `val['a']`라고 쓰면 에러 남. `json.loads(val)`를 한 번 더 해야 함. (귀찮음)

2. **`deserialize_json=True` (추천 )**
	- 결과: `{'a': 1}` (파이썬 딕셔너리)
	- 장점: 바로 `val['a']`라고 쓰면 값 `1`이 튀어나옴.

>**`default_var`** 도 실무에서 진짜 많이 써요.
>`Variable.get("my_key", default_var="없음")` 이렇게 짜면, 누가 실수로 UI에서 변수를 지워버려도 **DAG가 빨간불(Error)을 뿜으며 죽는 대참사**는 막을 수 있거든요. 😉


----
## 트러블슈팅: 환경변수 주입 삽질 기록

`daily_report.py` 안에서 DB 비밀번호 같은 민감 정보가 필요한데, 이걸 코드에 하드코딩하기 싫어서 여러 방법을 시도하다 막히는 패턴

### ❌ 실패 1: `env_file` 방식 (용량 에러)

```yaml
# docker-compose.yml
x-airflow-common: &airflow-common
  env_file:
    - .env   # ← 여기에 추가했을 때 에러 발생
```

**에러 내용:**
```
docker.errors.APIError: ... request body too large
```

**왜 터지나?**

`.env` 파일을 `x-airflow-common`에 넣으면, Airflow 컨테이너들이 뜰 때 `.env`의 모든 내용을 읽어서 환경변수로 등록하려 합니다.
이때 `.env` 파일 크기가 Docker API의 요청 허용 한도를 초과하면 터집니다.
```
.env 전체 내용
    ↓
Docker API로 전송 (컨테이너 생성 요청에 포함)
    ↓
💥 request body too large
```

>`.env` 파일에 내용이 많을수록, 또는 긴 값(긴 API 키, 긴 경로 등)이 있을수록 잘 발생합니다.


### 해결: `environment` + `Variable.get` 조합

DockerOperator의 `environment` 파라미터로 직접 주입하고, 민감 정보는 Airflow Variables에서 가져옵니다.

```python
# DAG 파일 (dags/daily_report_dag.py)
from airflow.models import Variable

run_spark_job = DockerOperator(
    ...
    environment={
        'DB_HOST': Variable.get("daily_report_db_host"),  # Airflow 금고에서 꺼냄
        'DB_PORT': '5432',
        'DB_NAME': 'airflow',
        'DB_USER': 'airflow',
        'DB_PASSWORD': Variable.get("daily_report_db_pwd")
    }
)
```

```python
# daily_report.py (컨테이너 안에서 실행)
import os
db_host = os.getenv("DB_HOST")  # ✅ 이제 값이 잘 들어옴!
```

**흐름:**
```
Airflow Variables (금고)
    ↓ Variable.get()
DAG 파일 environment={} 에 담김
    ↓ DockerOperator
새 컨테이너 환경변수로 주입
    ↓ os.getenv()
daily_report.py에서 정상 읽힘 ✅
```

|방법|결과|이유|
|---|---|---|
|`env_file=".env"`|❌ 용량 에러|Docker API body 크기 제한 초과|
|`os.getenv()` 단독|❌ None 반환|새 컨테이너에 환경변수 주입 안 됨|
|`environment` + `Variable.get()`|✅ 정상 작동|Variables→DAG→컨테이너 순서로 주입|

> `os.getenv` 자체가 틀린 게 아닙니다. `environment`로 먼저 주입해줘야 컨테이너 안에서 `os.getenv`로 읽을 수 있습니다. **주입 → 읽기** 순서가 핵심입니다.



---