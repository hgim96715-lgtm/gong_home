---
aliases:
  - jar
  - SQL
  - JDBC
  - JDBC Driver
  - spark-submit jars
  - spark.read.jdbc
  - write.jdbc
  - PostgreSQL Spark 연결
  - org.postgresql.Driver
tags:
  - Spark
  - PostgreSQL
related:
  - "[[Spark_Data_IO]]"
  - "[[Python_Database_Connect]]"
  - "[[00_Apache_Spark_HomePage]]"
---
# Spark JDBC — Spark 에서 PostgreSQL 읽고 쓰기

## 한 줄 요약

> **Spark 가 PostgreSQL 과 대화하려면 JDBC Driver(.jar) 라는 통역사 파일이 반드시 필요하다.**

---

---

# ① 핵심 개념 — 서버 vs 드라이버

```
가장 많이 하는 오해:
  "Docker 에 PostgreSQL 을 띄웠는데 왜 Spark 에서 접속이 안 되나요?"

  → DB 서버가 떠 있는 것과 드라이버가 있는 것은 완전히 다른 문제
```

|구분|역할|비유|
|---|---|---|
|PostgreSQL 컨테이너|데이터가 저장된 **서버**|와이파이 공유기 (신호를 쏘는 주체)|
|Spark 컨테이너|데이터를 처리하는 **클라이언트**|노트북 (접속하려는 주체)|
|JDBC Driver (.jar)|접속 규약이 담긴 **파일**|무선 랜카드 드라이버 (접속 방법)|

```
결론:
  PostgreSQL 서버가 아무리 잘 떠 있어도
  Spark 컨테이너 안에 드라이버 파일(.jar) 을 넣어주지 않으면
  절대 접속할 수 없다
```

---

---

# ② 네트워크 — 외부 접속 vs 내부 접속

```
접속 위치에 따라 host 와 port 가 달라진다.
```

```
외부 접속 (맥북 터미널, DataGrip, 로컬 파이썬)
  → 아파트 밖에서 외부 호출 벨을 누르는 것
  → host: localhost / port: 5433  (호스트 포트)

내부 접속 (Spark 컨테이너 → PostgreSQL 컨테이너)
  → 같은 아파트 안 이웃집에 내부 복도로 가는 것
  → host: postgres  (Docker 서비스 이름)
  → port: 5432      (컨테이너 내부 포트)
```

```python
# 외부 접속 (로컬에서 테스트할 때)
JDBC_URL = "jdbc:postgresql://localhost:5433/train_db"

# 내부 접속 (Spark 컨테이너에서 실행할 때) ← 이 프로젝트는 이쪽
JDBC_URL = "jdbc:postgresql://postgres:5432/train_db"
```

> 상세 → [[Docker_Networking]]

---

---

# ③ 환경 구축

## Step 1 — Driver(.jar) 파일 다운로드

```bash
# 프로젝트 루트에서 실행
mkdir -p spark_drivers

curl -O --output-dir spark_drivers \
    https://jdbc.postgresql.org/download/postgresql-42.7.1.jar
```

```
폴더 위치:
seoul-train-realtime-project/
├── spark_drivers/
│   └── postgresql-42.7.1.jar   ← 여기에 저장
└── spark/
    └── consumer.py
```

## Step 2 — docker-compose.yml 에 볼륨 마운트

```yaml
services:

  spark-master:
    image: apache/spark:3.5.0
    container_name: train-spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "8080:8080"
      - "7077:7077"
    volumes:
      # 코드 폴더 마운트
      - ./spark:/opt/spark/apps
      # JDBC 드라이버 파일 마운트 ⭐️ 폴더째 연결 금지! 파일 대 파일로 연결
      - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
    networks:
      - train-network

  spark-worker:
    image: apache/spark:3.5.0
    container_name: train-spark-worker
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    environment:
      - SPARK_WORKER_MEMORY=1G
      - SPARK_WORKER_CORES=1
    volumes:
      # 워커도 반드시 동일하게 마운트해야 함
      - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
    depends_on:
      - spark-master
    networks:
      - train-network
```

```
⚠️ 폴더째 마운트 금지

  # ❌ 이렇게 하면 안 됨
  - ./spark_drivers:/opt/spark/jars

  → /opt/spark/jars 안에 Spark 기본 엔진 파일들이 있음
  → 폴더째 덮어씌우면 기본 파일이 사라져서 Spark 자체가 망가짐

  # ✅ 파일 하나만 정확히 지정
  - ./spark_drivers/postgresql-42.7.1.jar:/opt/spark/jars/postgresql-42.7.1.jar
```

---

---

# ④ Python 코드 — JDBC 연결 4단계

## 1단계 — 접속 정보 (.env)

```bash
# .env
POSTGRES_USER=train_user
POSTGRES_PASSWORD=train_password
POSTGRES_DB=train_db
```

## 2단계 — JDBC URL 조립

```python
import os
from dotenv import load_dotenv

load_dotenv()

PG_HOST = "postgres"                          # Docker 내부 서비스 이름
PG_PORT = "5432"                              # 내부 포트
PG_DB   = os.getenv("POSTGRES_DB", "train_db")

JDBC_URL = f"jdbc:postgresql://{PG_HOST}:{PG_PORT}/{PG_DB}"
# 결과: "jdbc:postgresql://postgres:5432/train_db"
```

## 3단계 — 드라이버 지정 (핵심)

```python
JDBC_PROPS = {
    "user":     os.getenv("POSTGRES_USER"),
    "password": os.getenv("POSTGRES_PASSWORD"),
    "driver":   "org.postgresql.Driver",   # ⭐️ 이 줄 없으면 에러
}
```

```
"driver" 가 없으면:
  Spark → "어떤 언어로 말해야 해?" → ClassNotFoundException 에러

"org.postgresql.Driver" 는
  아까 마운트한 postgresql-42.7.1.jar 파일 안에 들어있는 클래스 이름
```

## 4단계 — 읽기 / 쓰기

### (읽기) read.jdbc — DB → DataFrame


```python
df = spark.read.jdbc(
    url=JDBC_URL,           # 접속 주소
    table="train_delay",    # 가져올 테이블
    properties=JDBC_PROPS,  # 인증 + 드라이버
)
df.show()
df.printSchema()
```

```
spark.read.jdbc 주요 파라미터:

  url        → JDBC 접속 주소  "jdbc:postgresql://postgres:5432/train_db"
  table      → 테이블명 또는 서브쿼리
  properties → 딕셔너리 {"user", "password", "driver"}

table 에 서브쿼리도 가능 (괄호 + 별명 필수):
  table="(SELECT * FROM train_delay WHERE dep_delay > 5) AS sub"
```

python

```python
# 특정 조건만 읽어오기 — 서브쿼리 방식
df = spark.read.jdbc(
    url=JDBC_URL,
    table="(SELECT * FROM train_delay WHERE dep_delay > 5) AS sub",
    properties=JDBC_PROPS,
)
```

---

### (쓰기) write.jdbc — DataFrame → DB

```python
df.write.jdbc(
    url=JDBC_URL,           # 접속 주소
    table="train_delay",    # 저장할 테이블
    mode="append",          # 저장 방식
    properties=JDBC_PROPS,  # 인증 + 드라이버
)
```

```
mode 4가지:

  "append"    → 기존 데이터 유지 + 새 데이터 추가
                스트리밍 적재 / 로그 누적에 사용

  "overwrite" → 테이블 전체 삭제 후 새로 씀
                배치로 전체 재적재할 때 사용
                ⚠️ Streaming 에서 사용 금지 — 배치마다 데이터가 사라짐

  "ignore"    → 테이블이 이미 있으면 아무것도 안 함
                초기 세팅 / 중복 방지

  "error"     → 테이블이 이미 있으면 에러 발생 (기본값)
                실수로 덮어쓰는 것을 방지
```

---

### Streaming 에서 write.jdbc — foreachBatch 필수

```
Streaming DataFrame 은 write.jdbc() 를 직접 쓸 수 없다.

  ❌ result_df.write.jdbc(...)
     → AnalysisException: Queries with streaming sources must be executed with writeStream

이유:
  Streaming DataFrame 은 데이터가 계속 흘러오는 상태
  "지금 전체를 저장해" 라는 명령을 내릴 수 없음

해결:
  foreachBatch 로 trigger 주기마다 잘라서 일반 DataFrame 으로 받아야 함
  그 안에서 write.jdbc() 호출
```


```python
def write_batch(batch_df, batch_id):
    if batch_df.isEmpty():    # 빈 배치도 호출되므로 체크 필수
        return
    batch_df.write.jdbc(      # 일반 DataFrame 이므로 write.jdbc 가능
        url=JDBC_URL,
        table="train_delay",
        mode="append",
        properties=JDBC_PROPS,
    )

query = (
    result_df                          # Streaming DataFrame
    .writeStream
    .foreachBatch(write_batch)         # 배치마다 write_batch 호출
    .option("checkpointLocation", "/tmp/checkpoint/delay")
    .trigger(processingTime="30 seconds")
    .start()
)
```

```
read.jdbc vs write.jdbc 비교:

  spark.read.jdbc(...)        → DB 에서 DataFrame 으로 읽기  (배치 전용)
  batch_df.write.jdbc(...)    → DataFrame 을 DB 에 저장      (배치 / foreachBatch 안)

  Streaming 에서 쓰기:
    writeStream.foreachBatch(함수) 안에서 write.jdbc 호출
    함수 시그니처는 반드시 (batch_df, batch_id) 2개 인자
```

---

---

# ⑤ spark-submit 실행

```bash
# 컨테이너 내부에서 직접 실행
docker exec -it train-spark-master \
  /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/consumer.py
```

```
--jars 옵션
  볼륨 마운트로 이미 /opt/spark/jars 에 있지만
  명시적으로 넣어주면 더 안전함
  여러 개면 콤마로 구분: --jars a.jar,b.jar

--master spark://spark-master:7077
  Standalone 클러스터 모드 (마스터에게 작업 제출)
  로컬 테스트: --master local[2]
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`ClassNotFoundException: org.postgresql.Driver`|.jar 파일 마운트 안 됨|docker-compose.yml 볼륨 마운트 확인|
|`Connection refused`|host/port 오류|컨테이너 내부면 `postgres:5432`, 외부면 `localhost:5433`|
|Spark 컨테이너가 시작하자마자 종료|jars 폴더 통째 마운트로 엔진 파일 덮어씌워짐|파일 대 파일로 마운트 (`파일.jar:/opt/spark/jars/파일.jar`)|
|`WARN TaskSchedulerImpl: No executors`|Worker 가 Master 에 연결 안 됨|`docker compose ps` 로 spark-worker 상태 확인|
|`--packages` 사용 시 느림|실행할 때마다 Maven 에서 다운로드|.jar 미리 받아서 볼륨 마운트 방식 권장|