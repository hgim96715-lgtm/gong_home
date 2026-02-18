---
aliases:
  - jar
  - SQL
tags:
  - Spark
related:
  - "[[Spark_Data_IO]]"
  - "[[Python_Database_Connect]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한줄 요약

 Spark가 PostgreSQL(또는 다른 RDB)과 대화하려면 **JDBC Driver(.jar)** 라는 '통역사'가 반드시 필요하다.

---
## 개념 잡기: 서버(Server) vs 드라이버(Driver)

가장 많이 하는 오해: **"Docker에 DB를 설치했는데 왜 접속이 안 되나요?"**

|**구분**|**역할**|**비유**|
|---|---|---|
|**PostgreSQL Container**|데이터가 저장된 **서버**|**와이파이 공유기** (신호를 쏘는 주체)|
|**Spark Container**|데이터를 분석하는 **클라이언트**|**노트북** (접속하려는 주체)|
|**JDBC Driver (.jar)**|접속 규약이 담긴 **파일**|**무선 랜카드 드라이버** (접속 방법)|

**결론:** Docker에 DB 서버(공유기)가 아무리 잘 떠 있어도, Spark 컨테이너 안에 드라이버 파일(랜카드)을 넣어주지 않으면 **절대 접속할 수 없다.**

---
## 네트워크 원리: 외부 문 vs 내부 복도

도커 컨테이너는 하나의 **"독립된 아파트"** 와 같습니다. 접속하는 위치에 따라 주소가 달라집니다.

- **외부 접속 (맥북에서 접속할 때):** 아파트 **"밖"** 에 있는 상태입니다. 
- 관리인이 열어준 외부 호출 벨 번호인 **`5433`** 을 누르고, 주소는 내 컴퓨터인 **`localhost`** 를 사용합니다. (예: DBeaver, 로컬 파이썬 실행)
- **내부 접속 (Spark 컨테이너에서 접속할 때):** 이제 Spark와 Postgres는 같은 아파트 **"안"에 사는 이웃입니다. 굳이 밖으로 나갈 필요 없이 내부 복도를 통해 옆집 벨(`5432`**)을 누르면 됩니다. 이때 주소는 서비스 이름인 **`postgres`** 가 됩니다.

>[[Docker_Host_vs_Internal_Network]] 참고


---
## 환경 구축 (Setup)

### Step 1. 드라이버 파일 준비 (.jar)

Spark가 읽을 수 있는 로컬 폴더를 만들고 드라이버를 다운로드한다.

```bash
# 1. 관리용 폴더 생성
mkdir -p spark_drivers

# 2. PostgreSQL JDBC Driver 다운로드 (버전은 상황에 맞게 변경)
cd spark_drivers
curl -O https://jdbc.postgresql.org/download/postgresql-42.7.1.jar
```

### Step 2. Docker 볼륨 연결 (`docker-compose.yaml`)

내 컴퓨터에 있는 드라이버 파일을 Spark 컨테이너의 라이브러리 폴더(`/opt/spark/jars`)로 **마운트(주입)** 해준다.

```yaml
services:
  spark-master:
    # ... (기본 설정) ...
    env_file: .env # 👈 [1] .env의 모든 변수(USER, PASS 등)를 일단 다 가져옴
    environment:
      - DB_HOST=postgres # 👈 [2] .env에 'localhost'라고 되어있어도 'postgres'로 덮어씀!
      - DB_PORT=5432     # 👈 [3] '5433' 대신 내부 포트인 '5432'로 덮어씀!
    volumes:
      - ./spark_app:/opt/spark/apps
      # [주의] 폴더째 연결 금지! '파일 대 파일'로 연결해서 Spark 엔진 파일을 보호함
      - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
    networks:
      - data-network

  spark-worker:
    # ... (기본 설정) ...
    volumes:
      - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
    networks:
      - data-network
```

---
## 1단계: 접속 정보 분리하기 (Configuration)

코드 안에 아이디/비번을 직접 적으면(하드코딩), 나중에 DB 비번이 바뀔 때마다 코드를 다 뜯어고쳐야 합니다.
`.env`에 정리 하거니 docker에 저장 

```.env
# [설정 영역] 여기만 수정하면 모든 DB에 접속 가능합니다.
DB_HOST = "postgres"       # 접속할 주소 (Docker 서비스명)
DB_PORT = "5432"           # 포트 번호
DB_NAME = "airflow"        # 데이터베이스 이름
DB_USER = "airflow"        # 아이디
DB_PASS = "airflow"        # 비밀번호
```

---
### 2단계: 주소 조립하기 (JDBC URL)

Spark가 DB를 찾아가려면 **정확한 주소 포맷(URL)** 이 필요합니다. 
PostgreSQL은 `jdbc:postgresql://주소:포트/DB이름`이라는 규칙을 씁니다

```python
# f-string을 사용해 위에서 설정한 변수들로 주소를 완성합니다.
# 결과 예시: "jdbc:postgresql://postgres:5432/airflow"
jdbc_url = f"jdbc:postgresql://{DB_HOST}:{DB_PORT}/{DB_NAME}"
```

### 3단계: 통역사 지정하기 (Driver Class) ⭐️ **가장 중요!**

주소만 안다고 대화가 통하지 않습니다. 
**"나는 PostgreSQL 언어를 쓸 거야"** 라고 Spark에게 명확히 알려줘야 합니다.

```python
db_properties = {
    "user": os.getenv(DB_USER),
    "password": os.getenv(DB_PASS),
    # [핵심] 이 줄이 없으면 Spark는 "어? 나 무슨 언어로 말해야 해?" 하고 에러를 냅니다.
    # 아까 다운로드 받은 .jar 파일이 이 역할을 수행합니다.
    "driver": "org.postgresql.Driver" 
}
```

### 4단계: 데이터 읽어오기 (Action)

이제 준비된 주소(`url`)와 통역사(`properties`)를 데리고 실제로 데이터를 가져옵니다.

```python
# spark.read.jdbc() 함수가 실제 연결을 시도합니다.
df = spark.read.jdbc(
    url=jdbc_url, 
    table="user_logs",   # 가져올 테이블 이름
    properties=db_properties
)
```



---
## 실행 명령어 (spark-submit)

실행할 때도 `--jars` 옵션으로 드라이버 위치를 명시적으로 알려주는 것이 안전하다.

```bash
# 컨테이너 내부 경로 기준 (/opt/spark/jars/...)
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/my_script.py
```

