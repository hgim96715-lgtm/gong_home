---
aliases:
  - Airflow Provider
  - 프로바이더
  - 외부 연동
  - 패키지 설치
  - Modular Architecture
tags:
  - Airflow
  - 확장성
  - Provider
related:
  - "[[Airflow_Operators]]"
  - "[[Docker_Host_Access]]"
---
## 개념 한 줄 요약

Airflow라는 '기본 몸체'에 **AWS, Postgres, Slack 같은 외부 서비스와 연동할 수 있는 기능(Operator, Sensor, Hook)을 추가해주는 '확장 패키지'** 입니다.

---
## 왜 필요한가 (Why)

**문제 상황:**

* Airflow를 처음 설치하면 정말 **'기본 기능(Core)'** 만 들어있습니다.
* "AWS S3에 파일 올려줘"라고 시키면 Airflow는 "AWS가 뭔데? 난 모르는 애야"라고 에러를 뱉습니다.
* 모든 기능(AWS, GCP, Azure, Slack 등)을 처음부터 다 넣으면 Airflow가 너무 무거워지기 때문입니다.

**해결책 (Modular Architecture):**
* 필요한 기능만 **'골라서 설치(Pip install)'** 할 수 있게 모듈화했습니다.
* AWS를 쓰고 싶으면 `amazon-provider`를 깔고, DB를 쓰고 싶으면 `postgres-provider`를 까는 식입니다.

---
## 실무 적용 3단계 프로세스 (How-to)

### 1단계: 설치 (Installation)

가장 먼저 파이썬 패키지를 설치해야 합니다. 
도커 환경에서는 `requirements.txt`에 추가하거나 이미지를 다시 빌드해야 합니다.

```bash
# 예시: Postgres 연동 기능 설치
pip install apache-airflow-providers-postgres
```

>[install 확인](https://airflow.apache.org/docs/apache-airflow-providers/packages-ref.html#apache-airflow-providers-postgres)
>나는  Docker 로 하고 있으니깐 밑에 도커(Docker) 환경에서 영구적으로 설치하는 법 (중요!) 이걸로 설치 하고 다시 2단계 

### 2단계: 연결 설정 (Connection UI)

패키지를 깔면 웹 UI의 **Admin -> Connections** 메뉴에 새로운 'Connection Type'이 생깁니다. 여기에 아이디/비밀번호를 등록합니다.

- **Conn Id:** 코드에서 부를 별명 (예: `my_postgres_connection`)
- **Conn Type:** `Postgres` (프로바이더를 안 깔면 이 항목이 없음!)
- **Host/Login/Password:** DB 접속 정보 입력

| **항목 (Field)**      | **입력값 (Value)**          | **이유/설명**                               |
| ------------------- | ------------------------ | --------------------------------------- |
| **Connection Id**   | `my_postgres_connection` | 코드에서 부를 이름 (내 마음대로)                     |
| **Connection Type** | `Postgres`               | 목록에 없으면 Provider 설치 안 된 것임              |
| **Host**            | **`postgres`**           | 같은 도커 네트워크 안에 있으니 **서비스 이름**으로 부르면 됩니다. |
| **Schema**          | `postgres`               | `POSTGRES_DATABASE` 값                   |
| **Login**           | `airflow`                | `POSTGRES_USER` 값                       |
| **Password**        | `airflow`                | `POSTGRES_PASSWORD` 값                   |
| **Port**            | `5432`                   | 기본 포트                                   |

### 3단계: 코드 작성 (DAG Development)

이제 코드에서 해당 프로바이더가 제공하는 **전용 오퍼레이터(Operator)** 를 가져다 쓸 수 있습니다.

```python
# 1. 공통 SQL 프로바이더에서 범용 일꾼을 데려옵니다.
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

with DAG(...) as dag:
    # 2. 어떤 SQL이든 실행할 수 있는 오퍼레이터를 생성합니다.
    execute_task = SQLExecuteQueryOperator(
        # 태스크의 이름 (DataGrip에서 본 그 테이블을 건드릴 이름)
        task_id="create_my_table",
        
        # [중요] 웹 UI(Admin -> Connections)에서 만든 연결 아이디를 적습니다.
        # 예전처럼 postgres_conn_id라고 쓰면 에러가 날 수 있으니 주의!
        conn_id="my_postgres_connection",
        
        # 실행할 SQL 문을 적습니다. (여러 줄은 따옴표 3개 활용)
        sql="""
            CREATE TABLE IF NOT EXISTS test_table (
                id SERIAL PRIMARY KEY,
                name TEXT
            );
        """
    )
```

- **임포트 경로 변경**: `providers.postgres`가 아니라 **`providers.common.sql`** 에서 가져옵니다.
- **매개변수 통합**: 특정 DB 이름을 딴 `postgres_conn_id` 대신, 모든 DB에 공통으로 쓰이는 **`conn_id`** 라는 이름을 사용합니다.

### 4단계 : 확인법 airflow tasks test

```bash
# 1. 테이블 생성 테스트
docker compose run airflow-scheduler airflow tasks test postgres_loader create_sample_table 2026-01-01

# 2. 데이터 삽입 테스트
docker compose run airflow-scheduler airflow tasks test postgres_loader insert_sample_data 2026-01-01
```

- **`airflow tasks test`** 는 DAG 전체를 돌리지 않고, **특정 태스크 하나만 콕 집어서 내 컴퓨터에서 즉시 실행**해볼 수 있는 '단위 테스트' 명령어입니다.

### table로 확인법 -> DataGrip

- 테이블을 생성 Postgres로 :, Docker환경이라면 , airflow로 해야한다! 

> 이때, 원래 5432였는데 계속 오류가 났다 그 이유가 5432번 연결이 로컬 DB와의 충돌 로 오류 남
> **포트 포워딩 (`5433:5432`)**: 맥북의 5433번 구멍을 도커의 5432번 구멍으로 연결하여 로컬 DB와의 충돌을 피했습니다.
> docker-compose.yaml 에서 postgres의 posts를 "5433:5432" 로 변경 

![[스크린샷 2026-01-21 오후 6.49.07.png|500x400]]

---
##  도커(Docker) 환경에서 영구적으로 설치하는 법 (중요!)

> "터미널에서 `pip install` 했는데, 재시작하니까 사라졌어요!"

### 1. 왜 사라지나요? (The Problem) 

Docker 컨테이너는 **'일회용 도시락'** 과 같습니다. 
실행 중에 설치한 패키지는 컨테이너를 끄는 순간(`docker compose down`) 초기화되어 사라집니다. 
따라서 **'반찬(라이브러리)이 미리 담겨있는 나만의 도시락(Custom Image)'** 을 만들어야 합니다.

### 2. 해결책: 커스텀 이미지 만들기 (Build)

**Step 1: `requirements.txt` 만들기**

`dags` 폴더가 있는 최상위 경로에 파일을 만들고 필요한 라이브러리를 적습니다.

```text
apache-airflow-providers-common-sql 
apache-airflow-providers-postgres
```

**Step 2: `Dockerfile` 만들기** 

마찬가지로 최상위 경로에 `Dockerfile`이라는 이름의 파일(확장자 없음)을 만들고 아래 내용을 적습니다.

```Dockerfile
# 1. 공식 Airflow 이미지를 베이스로 사용
FROM apache/airflow:2.8.1

# 2. requirements.txt 파일을 컨테이너 안으로 복사
COPY requirements.txt /requirements.txt

# 3. pip install 실행 (캐시 없이 가볍게)
RUN pip install --no-cache-dir -r /requirements.txt
```

**Step 3: 다시 빌드하고 실행하기** 

터미널에서 아래 명령어를 입력하면, 새로운 라이브러리가 포함된 이미지가 구워집니다.

```bash
docker compose build    # 이미지 빌드 (오래 걸림)
docker compose up -d    # 실행
```

---
## 핵심 구성 요소

프로바이더를 설치하면 보통 다음 3가지 세트가 같이 들어옵니다.

1. **Operator:** `PostgresOperator`처럼 작업을 수행하는 도구.
2. **Hook:** 외부 시스템과 통신하는 실제 연결 통로 (보통 Operator 내부에서 쓰임).
3. **Sensor:** `S3KeySensor`처럼 외부 시스템의 상태를 감시하는 도구.

---
## 초보자가 자주 착각하는 포인트

1. **"설치(`pip install`) 안 하고 코드(`import`)만 썼어요."**
    - `ModuleNotFoundError`가 발생합니다. 
    - 반드시 설치가 선행되어야 합니다.

2. **"Connection ID를 내 마음대로 지어도 되나요?"**
    - 네, 마음대로 지어도 됩니다.
    - 단, **웹 UI에 적은 ID**와 **파이썬 코드에 적은 ID**가 **토씨 하나 안 틀리고 똑같아야** 연결됩니다.

3. **그냥 `docker exec`로 들어가서 설치하면 안 되나요?**
	- 테스트용으로는 괜찮습니다. 
	- 하지만 컨테이너가 죽거나 업데이트할 때마다 매번 다시 설치해야 하므로, **프로덕션(운영) 환경**에서는 반드시 위처럼 `Dockerfile`을 만들어야 합니다.
