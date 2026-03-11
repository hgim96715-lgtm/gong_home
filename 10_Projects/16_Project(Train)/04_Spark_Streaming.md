---
aliases:
  - Spark Streaming
  - PySpark
  - Structured Streaming
  - readStream
  - writeStream
  - Kafka Spark
  - spark-sql-kafka
tags:
  - Project
related:
  - "[[00_Seoul Station Real-Time Train Project|train-project]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[03_Kafka_Producer]]"
  - "[[Kafka_Python_Consumer]]"
  - "[[Spark_JDBC_PostgreSQL]]"
  - "[[Spark_DataFrame]]"
  - "[[Spark_Core_Objects]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
  - "[[DataFrame_Transform]]"
  - "[[Spark_JSON_Handling]]"
---
# 04_Spark_Streaming — Kafka 데이터를 받아서 PostgreSQL 에 저장하기

## 이 단계의 목표

```
STEP 3 에서 Kafka 토픽에 쌓인 데이터를
Spark Structured Streaming 으로 실시간 소비하여
PostgreSQL 에 저장하는 것까지

train-schedule  토픽 → train_schedule  테이블
train-realtime  토픽 → train_realtime  테이블
train-delay     토픽 → train_delay     테이블
```

---

---

# ① Docker Compose 에 Spark 추가

> [[01_Docker_Setup_Postgresql_Setup]] 의 `docker-compose.yml` 에 아래 서비스를 추가한다.

```text
⚠️ bitnami/spark 사용 불가
   2025년 9월 이후 Bitnami 이미지 유료 전환 (Broadcom 인수)
   → apache/spark:3.5.0 공식 이미지 사용

bitnami vs apache 공식 이미지 차이:
  bitnami → SPARK_MODE, SPARK_MASTER_URL 환경변수로 역할 지정
  apache  → command 로 직접 클래스 실행
    spark-class ...Master              → 마스터 역할
    spark-class ...Worker spark://...  → 워커 역할

networks: train-network 반드시 정의 필요
  없으면 "refers to undefined network" 에러 발생
  docker-compose.yml 맨 아래 networks: / train-network: / driver: bridge 추가
```

```yaml
# docker-compose.yml 에 추가
  spark-master:
    image: apache/spark:3.5.0
    container_name: train-spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "8080:8080"   # Spark Web UI
      - "7077:7077"   # Spark Master 포트
    networks:
      - train-network

  spark-worker:
    image: apache/spark:3.5.0
    container_name: train-spark-worker
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    environment:
      - SPARK_WORKER_MEMORY=1G
      - SPARK_WORKER_CORES=1
    depends_on:
      - spark-master
    networks:
      - train-network
        
networks:
  train-network:
    driver: bridge
```

```
bitnami vs apache 공식 이미지 차이:

  bitnami
    SPARK_MODE=master        → 환경변수로 역할 지정
    SPARK_MASTER_URL=...     → 환경변수로 마스터 주소 지정
    SPARK_RPC_* 환경변수 사용

  apache 공식
    command 로 직접 클래스 실행
    spark-class ...Master    → 마스터 역할
    spark-class ...Worker spark://host:port  → 워커 역할
    환경변수 SPARK_WORKER_MEMORY / SPARK_WORKER_CORES 는 동일하게 사용 가능

Spark Web UI 확인:
  http://localhost:8080
  → Workers, Running Applications 상태 확인
```

---

---

# ② 설계 구조

```
Kafka 토픽 3개
  train-schedule  ─┐
  train-realtime  ─┤─→ Spark Structured Streaming
  train-delay     ─┘         ↓
                        JSON 파싱 + 스키마 적용
                             ↓
                        foreachBatch
                             ↓
                        JDBC (PostgreSQL)
                             ↓
                     train_schedule / train_realtime / train_delay 테이블
```

```
consumer.py 구조:
  TrainConsumer (클래스)
  ├── __init__()           SparkSession 생성 + JDBC 설정
  ├── _read_stream()       Kafka 토픽 → DataFrame
  ├── _write_jdbc()        DataFrame → PostgreSQL (foreachBatch)
  ├── run_schedule()       train-schedule 토픽 처리
  ├── run_realtime()       train-realtime 토픽 처리
  ├── run_delay()          train-delay 토픽 처리
  └── run()                스트림 3개 동시 실행 + awaitAnyTermination
```

---

---

# ③ 파일 구조 및 의존성

## 파일 위치

```
seoul-train-realtime-project/
└── spark/
    ├── consumer.py       ← STEP 4 (Spark Consumer) ← 여기
    └── requirements.txt
```

## requirements.txt

```
pyspark==3.5.0
python-dotenv==1.0.0
```


```bash
pip3 install -r spark/requirements.txt
```

## Spark 패키지 (실행 시 자동 다운로드)

```
Kafka 연동 패키지:
  org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0

PostgreSQL JDBC 드라이버:
  org.postgresql:postgresql:42.7.3

→ spark-submit --packages 옵션으로 넘겨줌 (아래 실행 섹션 참고)
```

## .env 확인

```bash
# .env
POSTGRES_USER=train_user
POSTGRES_PASSWORD=train_password
POSTGRES_DB=train_db
KAFKA_BOOTSTRAP_SERVERS=kafka:9092   # producer/consumer 모두 통일
```

## /etc/hosts 설정 — 맥북에서 kafka 이름 인식

```text
producer.py 는 맥북에서 직접 실행 → "kafka" 라는 이름을 모름
consumer.py 는 컨테이너 안에서 실행 → Docker 서비스명으로 kafka 인식

둘 다 KAFKA_BOOTSTRAP_SERVERS=kafka:9092 로 통일하려면
맥북 /etc/hosts 에 kafka = 127.0.0.1 로 등록 필요
```

```bash
# 1. /etc/hosts 열기 (맥북 비밀번호 요구)
sudo nano /etc/hosts

# 2. 맨 아랫줄에 추가
127.0.0.1 kafka

# 3. 저장 후 종료
#    Ctrl + O → Enter (저장)
#    Ctrl + X (종료)
```

```text
등록 후:
  맥북에서  kafka:9092 → 127.0.0.1:9092 (localhost) 로 해석
  컨테이너에서 kafka:9092 → Docker 서비스명 kafka 로 해석
  → .env 하나로 producer/consumer 모두 동일하게 사용 가능
```

---

---

# ④ Spark Consumer 코드 — consumer.py

```python
import os
from pyspark.sql import SparkSession, DataFrame
from pyspark.sql.functions import col, from_json, current_timestamp
from pyspark.sql.types import (
    StructType, StructField,
    StringType, IntegerType,
)

# docker-compose env_file 로 주입되므로 dotenv 없어도 됨
# 로컬 직접 실행 시를 위한 방어 처리
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass  # 컨테이너 안에서는 os.getenv() 만으로 충분

# =====================================================================
# 환경 변수 — docker-compose env_file: .env 로 자동 주입됨
# =====================================================================
KAFKA_BOOTSTRAP = os.getenv("KAFKA_BOOTSTRAP_SERVERS", "kafka:9092")
PG_USER         = os.getenv("POSTGRES_USER",     "train_user")
PG_PASSWORD     = os.getenv("POSTGRES_PASSWORD", "train_password")
PG_DB           = os.getenv("POSTGRES_DB",       "train_db")
PG_HOST         = "postgres"   # Docker 서비스명 — localhost ❌ (컨테이너 밖 주소)
PG_PORT         = "5432"       # 컨테이너 내부 포트 — 5433 ❌ (그건 맥북 접속 포트)

JDBC_URL = f"jdbc:postgresql://{PG_HOST}:{PG_PORT}/{PG_DB}"
JDBC_PROPS = {
    "user":     PG_USER,
    "password": PG_PASSWORD,
    "driver":   "org.postgresql.Driver",  # spark_drivers/postgresql-42.7.1.jar 안의 클래스
}

# =====================================================================
# 스키마 — Kafka JSON 구조와 1:1 매칭
# Producer 가 보내는 키 이름과 정확히 일치해야 함 (다르면 null)
# =====================================================================

SCHEMA_SCHEDULE = StructType([
    StructField("run_ymd",           StringType()),
    StructField("trn_no",            StringType()),
    StructField("dptre_stn_cd",      StringType()),
    StructField("dptre_stn_nm",      StringType()),
    StructField("arvl_stn_cd",       StringType()),
    StructField("arvl_stn_nm",       StringType()),
    StructField("trn_plan_dptre_dt", StringType()),  # VARCHAR(50) 으로 수정 필요 (기본 20 초과)
    StructField("trn_plan_arvl_dt",  StringType()),  # VARCHAR(50) 으로 수정 필요
    StructField("data_type",         StringType()),
    StructField("created_at",        StringType()),  # Spark 에서 current_timestamp() 로 덮어씌움
])

SCHEMA_REALTIME = StructType([
    StructField("trn_no",       StringType()),
    StructField("mrnt_nm",      StringType()),
    StructField("dptre_stn_nm", StringType()),
    StructField("arvl_stn_nm",  StringType()),
    StructField("plan_dep",     StringType()),
    StructField("plan_arr",     StringType()),
    StructField("status",       StringType()),
    StructField("progress_pct", IntegerType()),
    StructField("data_type",    StringType()),
    StructField("created_at",   StringType()),
])

SCHEMA_DELAY = StructType([
    StructField("run_ymd",      StringType()),
    StructField("trn_no",       StringType()),
    StructField("mrnt_nm",      StringType()),
    StructField("dptre_stn_nm", StringType()),
    StructField("arvl_stn_nm",  StringType()),
    StructField("plan_dep",     StringType()),
    StructField("plan_arr",     StringType()),
    StructField("real_dep",     StringType()),
    StructField("real_arr",     StringType()),
    StructField("dep_delay",    IntegerType()),
    StructField("arr_delay",    IntegerType()),
    StructField("dep_status",   StringType()),
    StructField("arr_status",   StringType()),
    StructField("data_type",    StringType()),
    StructField("created_at",   StringType()),
])

# =====================================================================
# TrainConsumer 클래스
# =====================================================================

class TrainConsumer:

    def __init__(self):
        self.spark = (
            SparkSession.builder
            .appName("TrainConsumer")
            .config("spark.sql.shuffle.partitions", "2")  # 기본값 200 → 소규모 프로젝트 최적화
            .getOrCreate()
        )
        self.spark.sparkContext.setLogLevel("WARN")  # INFO 로그 억제

    def _read_stream(self, topic: str) -> DataFrame:
        """Kafka 토픽 → Streaming DataFrame"""
        return (
            self.spark.readStream
            .format("kafka")
            .option("kafka.bootstrap.servers", KAFKA_BOOTSTRAP)  # kafka:9092 (서비스명)
            .option("subscribe", topic)
            .option("startingOffsets", "earliest")  # 토픽에 쌓인 메시지 처음부터 읽음
                                                    # checkpoint 있으면 무시되고 이어서 읽음
            .option("failOnDataLoss", "false")       # 오프셋 유실 시 중단 대신 경고
            .load()
        )

    def _write_jdbc(self, table: str):
        """클로저 패턴 — 테이블 이름을 기억하는 write_batch 함수 반환
        foreachBatch 는 (batch_df, batch_id) 2개 인자를 받는 함수만 허용
        → 테이블명을 바깥에서 주입하려면 클로저로 감싸야 함
        """
        def write_batch(batch_df, batch_id):
            if batch_df.isEmpty():  # 빈 배치도 호출되므로 체크 필수
                return
            batch_df.write.jdbc(
                url=JDBC_URL,
                table=table,
                mode="append",       # 기존 데이터 유지 + 추가 (overwrite 금지)
                properties=JDBC_PROPS,
            )
            print(f"[{table}] batch_id={batch_id} | {batch_df.count()}건 DB 저장 완료!")

        return write_batch  # 함수 자체를 반환 (호출 X)

    def run_schedule(self):
        """train-schedule 토픽 → train_schedule 테이블 (하루 1회 배치)"""
        raw = self._read_stream("train-schedule")

        parsed = (
            raw
            .select(from_json(col("value").cast("string"), SCHEMA_SCHEDULE).alias("data"))
            .select("data.*")
            .withColumn("created_at", current_timestamp())  # Producer 시각 대신 DB 저장 시각으로 덮어씌움
        )

        # DB train_schedule 컬럼과 정확히 일치 (id 는 SERIAL 이라 제외)
        final = parsed.select(
            "run_ymd", "trn_no",
            "dptre_stn_cd", "dptre_stn_nm",
            "arvl_stn_cd",  "arvl_stn_nm",
            "trn_plan_dptre_dt", "trn_plan_arvl_dt",
            "data_type", "created_at",
        )

        return (
            final.writeStream
            .foreachBatch(self._write_jdbc("train_schedule"))
            .option("checkpointLocation", "/tmp/checkpoint/schedule")  # 토픽마다 별도 경로
            .trigger(processingTime="10 seconds")
            .start()
        )

    def run_realtime(self):
        """train-realtime 토픽 → train_realtime 테이블 (60초마다 갱신)"""
        raw = self._read_stream("train-realtime")

        parsed = (
            raw
            .select(from_json(col("value").cast("string"), SCHEMA_REALTIME).alias("data"))
            .select("data.*")
            .withColumn("created_at", current_timestamp())
        )

        final = parsed.select(
            "trn_no", "mrnt_nm",
            "dptre_stn_nm", "arvl_stn_nm",
            "plan_dep", "plan_arr",
            "status", "progress_pct",
            "data_type", "created_at",
        )

        return (
            final.writeStream
            .foreachBatch(self._write_jdbc("train_realtime"))
            .option("checkpointLocation", "/tmp/checkpoint/realtime")
            .trigger(processingTime="10 seconds")
            .start()
        )

    def run_delay(self):
        """train-delay 토픽 → train_delay 테이블 (하루 1회, 새벽)"""
        raw = self._read_stream("train-delay")

        parsed = (
            raw
            .select(from_json(col("value").cast("string"), SCHEMA_DELAY).alias("data"))
            .select("data.*")
            .withColumn("created_at", current_timestamp())
        )

        final = parsed.select(
            "run_ymd", "trn_no", "mrnt_nm",
            "dptre_stn_nm", "arvl_stn_nm",
            "plan_dep", "plan_arr",
            "real_dep", "real_arr",
            "dep_delay", "arr_delay",
            "dep_status", "arr_status",
            "data_type", "created_at",
        )

        return (
            final.writeStream
            .foreachBatch(self._write_jdbc("train_delay"))
            .option("checkpointLocation", "/tmp/checkpoint/delay")
            .trigger(processingTime="30 seconds")  # 배치성 데이터 → 여유 있게 30초
            .start()
        )

    def run(self):
        """스트림 3개 동시 실행"""
        print("Spark Consumer 파이프라인 준비 완료!!!!")

        q_schedule = self.run_schedule()
        q_realtime = self.run_realtime()
        q_delay    = self.run_delay()

        print("[train-schedule] 모니터링 시작")
        print("[train-realtime] 모니터링 시작")
        print("[train-delay]    모니터링 시작")
        print("\n Spark Web UI 모니터링 → http://localhost:8080\n")

        # 셋 중 하나라도 에러로 종료되면 전체 종료
        self.spark.streams.awaitAnyTermination()


if __name__ == "__main__":
    tc = TrainConsumer()
    try:
        tc.run()
    except KeyboardInterrupt:
        print("\n 사용자의 요청으로 Consumer 종료")
    finally:
        tc.spark.stop()
```

---

---

# ⑤ Spark Structured Streaming 핵심 개념

## readStream — 스트림으로 읽기

```python
df = (
    spark.readStream
    .format("kafka")
    .option("subscribe", "train-realtime")       # 구독할 토픽
    .option("startingOffsets", "latest")          # latest: 이 시점 이후부터
    .load()                                       # earliest: 토픽 처음부터 전부
)
```

```
readStream 으로 읽은 DataFrame 은 "무한히 흘러오는 테이블" 로 이해
실제 데이터가 아직 없어도 에러 안 남 (Kafka 에 메시지 오면 처리)
```

## Kafka 메시지 구조

```
Kafka 에서 readStream 으로 읽으면 아래 컬럼이 자동으로 붙음

key       bytes    메시지 키
value     bytes    실제 데이터 (우리가 보낸 JSON)  ← 여기만 씀
topic     string   토픽 이름
partition int      파티션 번호
offset    long     파티션 내 위치
timestamp long     Kafka 기록 시각
```

```python
# value 를 문자열로 캐스팅 후 JSON 파싱
raw.select(
    from_json(col("value").cast("string"), SCHEMA).alias("data")
).select("data.*")
#           ↑
#   중첩 구조를 펼쳐서 컬럼으로 꺼냄
```

## foreachBatch — 배치 단위로 저장

```python
def write_batch(batch_df, batch_id):
    # batch_df   : trigger 주기마다 모인 미니 배치 DataFrame
    # batch_id   : 배치 번호 (0, 1, 2 ...)
    batch_df.write.jdbc(url=..., table=..., mode="append")

stream.writeStream
    .foreachBatch(write_batch)
    .start()
```

```
Structured Streaming 의 출력 방식:
  append       새 행만 출력 (우리 프로젝트는 이걸 씀)
  update       변경된 행만 출력
  complete     전체 결과 출력 (집계 쿼리에서 사용)

foreachBatch 를 쓰는 이유:
  JDBC 는 기본 sink 로 지원 안 함
  → foreachBatch 로 미니배치를 받아서 직접 write.jdbc() 호출
```

## trigger — 처리 주기

```python
.trigger(processingTime="10 seconds")   # 10초마다 미니배치 처리
.trigger(processingTime="1 minute")     # 1분마다
.trigger(once=True)                     # 딱 한 번만 처리하고 종료 (배치 모드)
```

```
processingTime 을 짧게:  실시간성 ↑, 리소스 소비 ↑
processingTime 을 길게:  리소스 ↓, 처리 지연 ↑

train-realtime  → 10초 (전광판용, 자주 갱신)
train-delay     → 30초 (하루 1회 배치, 여유 있게)
```

## checkpoint — 장애 복구

```python
.option("checkpointLocation", "/tmp/checkpoint/realtime")
```

```
Spark 가 "어디까지 처리했는지" 를 디스크에 저장해두는 것
Consumer 가 재시작되면 checkpoint 를 읽고 이어서 처리
없으면 재시작 시 중복 처리 또는 누락 발생 가능

반드시 토픽별로 다른 경로 사용 (같은 경로 쓰면 충돌)
  /tmp/checkpoint/schedule
  /tmp/checkpoint/realtime
  /tmp/checkpoint/delay
```

## awaitAnyTermination — 스트림 유지

```python
spark.streams.awaitAnyTermination()
```

```
스트림은 start() 하는 순간 백그라운드 스레드에서 돌아감
awaitAnyTermination() 없으면 메인 스레드가 바로 끝나버림
→ 스트림도 같이 죽음

awaitAnyTermination():
  등록된 스트림 중 하나라도 종료될 때까지 메인 스레드 대기
  에러로 인해 하나가 죽으면 나머지도 같이 종료
```

---

---

# ⑥ 실행

## 실행 순서

```bash
# 1. Kafka + PostgreSQL + Spark 컨테이너 실행
docker compose up -d

# 2. 컨테이너 상태 확인
docker compose ps

# 3. Producer 실행 (STEP 3)
cd producer
python3 producer.py &

# 4. Consumer 실행 (STEP 4)
# jar 파일을 컨테이너에 먼저 복사
docker cp spark_drivers/. train-spark-master:/opt/spark/jars/

# spark-submit 실행
docker exec -it train-spark-master \
  /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/spark-sql-kafka-0-10_2.12-3.5.0.jar,\
/opt/spark/jars/spark-token-provider-kafka-0-10_2.12-3.5.0.jar,\
/opt/spark/jars/kafka-clients-3.4.1.jar,\
/opt/spark/jars/commons-pool2-2.11.1.jar,\
/opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/consumer.py
```

## ⚠️ docker compose down 후 체크리스트

```text
docker compose down 하면 컨테이너가 완전히 삭제됨
볼륨(kafka_data, postgres_data) 은 유지되지만
Kafka 는 컨테이너가 새로 뜨면서 클러스터 ID 충돌로 볼륨을 무시하고 초기화됨
→ 토픽이 사라질 수 있음

아래 두 가지는 반드시 다시 해야 함:
  ① jar 파일 재복사
  ② Kafka 토픽 재생성
```

```bash
# ① jar 파일 재복사 (컨테이너 새로 만들어지면 사라짐)
docker cp spark_drivers/. train-spark-master:/opt/spark/jars/

# ② Kafka 토픽 확인 — 없으면 재생성
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list

# 토픽이 없으면 생성
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic train-schedule --partitions 1 --replication-factor 1

docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic train-realtime --partitions 1 --replication-factor 1

docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic train-delay --partitions 1 --replication-factor 1
```

```text
KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true" 설정이 있으면
Producer 가 처음 메시지를 보낼 때 토픽이 자동 생성됨
→ 수동 생성 안 해도 되는 경우가 많음
→ 하지만 Consumer 가 먼저 구독을 시도하면 토픽 없음 에러 발생 가능
→ 확실하게 하려면 위 명령어로 미리 만들어두는 것이 안전
```

```text
jar 파일 전달 방식:
  volumes 파일-by-파일 마운트
    → Docker 가 파일 없으면 폴더로 만들어버림 ❌

  docker cp spark_drivers/. train-spark-master:/opt/spark/jars/
    → 로컬 spark_drivers/ 폴더의 파일들을 컨테이너 /opt/spark/jars/ 로 복사
    → 컨테이너 재시작 시 사라짐 (재시작하면 docker cp 다시 실행)
    → 개발 환경에서 가장 간단한 방법 ✅

  docker compose down 후 재시작 시:
    → docker cp 한 번만 다시 실행하면 됨
```

```text
--packages vs --jars:

  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0
    → 실행할 때마다 Maven 에서 다운로드 시도
    → /home/spark/.ivy2/cache 디렉토리 없으면 FileNotFoundException 에러
    → 네트워크 없으면 실패

  --jars /opt/spark/jars/spark-sql-kafka-0-10_2.12-3.5.0.jar,...
    → docker cp 로 미리 복사한 파일을 직접 지정
    → 다운로드 없음, 빠름 ✅

Kafka 연동에 필요한 jar 5개 (모두 필요):
  spark-sql-kafka-0-10_2.12-3.5.0.jar              ← Kafka 연동 핵심
  spark-token-provider-kafka-0-10_2.12-3.5.0.jar   ← Kafka 토큰 인증 (의존성)
  kafka-clients-3.4.1.jar                          ← Kafka 클라이언트 (의존성)
  commons-pool2-2.11.1.jar                         ← 커넥션 풀 (의존성)
  postgresql-42.7.1.jar                            ← PostgreSQL JDBC
```

## spark_drivers/ 폴더 구성 (curl 로 다운로드)

```bash
# 맥북 터미널에서 실행 (프로젝트 루트에서)
cd spark_drivers

# 다운로드 
curl -O https://repo1.maven.org/maven2/org/apache/spark/spark-sql-kafka-0-10_2.12/3.5.0/spark-sql-kafka-0-10_2.12-3.5.0.jar
curl -O https://repo1.maven.org/maven2/org/apache/spark/spark-token-provider-kafka-0-10_2.12/3.5.0/spark-token-provider-kafka-0-10_2.12-3.5.0.jar
curl -O https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/3.4.1/kafka-clients-3.4.1.jar
curl -O https://repo1.maven.org/maven2/org/apache/commons/commons-pool2/2.11.1/commons-pool2-2.11.1.jar
curl -O https://repo1.maven.org/maven2/org/postgresql/postgresql/42.7.1/postgresql-42.7.1.jar

# 확인
ls -l
# commons-pool2-2.11.1.jar
# kafka-clients-3.4.1.jar
# postgresql-42.7.1.jar
# spark-sql-kafka-0-10_2.12-3.5.0.jar
# spark-token-provider-kafka-0-10_2.12-3.5.0.jar
```

```
--packages 옵션:
  spark-sql-kafka 패키지 이름 해석
    org.apache.spark      → 조직
    spark-sql-kafka-0-10  → Kafka 0.10+ 용 커넥터
    _2.12                 → Scala 2.12 버전 (Spark 3.x 기준)
    :3.5.0                → Spark 버전과 맞춰야 함 ⚠️

--master spark://spark-master:7077:
  Standalone 클러스터 모드 (컨테이너 안에서 실행 시)
  로컬 테스트 시: --master local[2]

/opt/spark/apps/consumer.py:
  volumes 마운트된 경로 (로컬 ./spark/consumer.py 와 동일)
```

## 정상 실행 확인

```
[train-schedule] 모니터링 시작
[train-realtime] 모니터링 시작
[train-delay] 모니터링 시작

 Spark Web UI 모니터링 → http://localhost:8080

[train_schedule] batch_id=2| 724건 DB저장 완료!
[train_realtime] batch_id=2| 5건 DB저장 완료!
```

```
DataGrip 에서 확인:
  SELECT COUNT(*) FROM train_realtime;
  SELECT COUNT(*) FROM train_delay;
```

---

---

# 내가 만난 오류 들 ...트러블슈팅 많이 만났군..

| 증상                                                         | 원인                                 | 해결                                                                                |
| ---------------------------------------------------------- | ---------------------------------- | --------------------------------------------------------------------------------- |
| `zsh: command not found: spark-submit`                     | 맥북에 Spark 없음                       | `docker exec -it train-spark-master /opt/spark/bin/spark-submit ...`              |
| `Failed to find data source: kafka`                        | jar 복사 안 됨                         | `docker cp spark_drivers/. train-spark-master:/opt/spark/jars/` 후 재실행             |
| `ClassNotFoundException: org.postgresql.Driver`            | postgresql jar 없음                  | `docker cp` 로 `postgresql-42.7.1.jar` 복사 확인                                       |
| `ModuleNotFoundError: No module named 'dotenv'`            | 컨테이너에 패키지 없음                       | `docker-compose.yml` 에 `env_file: - .env` 추가 → `dotenv` 불필요                       |
| `AttributeError: 'TrainConsumer' has no attribute 'spark'` | `__init__` 실패 (SparkSession 생성 오류) | Spark 로그 확인 — jar 누락 또는 master 연결 실패가 원인                                          |
| `Connection to localhost:5433 refused` (PostgreSQL)        | JDBC URL 이 맥북 주소                   | `PG_HOST="postgres"`, `PG_PORT="5432"` 로 수정                                       |
| `Connection to localhost:9092 failed` (Kafka)              | ADVERTISED_LISTENERS 오류            | `docker-compose.yml` 에서 `KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092` 로 수정 |
| `train-schedule` 데이터 없음                                    | checkpoint 가 offset 기억             | `docker exec train-spark-master rm -rf /tmp/checkpoint/` 후 재실행                    |
| 오타 토픽 생성 (`train-scedule` 등)                               | Producer 코드 오타                     | `kafka-topics.sh --delete --topic train-scedule` 로 삭제                             |
| `value too long for type character varying(20)`            | 시간 컬럼 길이 초과                        | `ALTER TABLE train_schedule ALTER COLUMN trn_plan_dptre_dt TYPE VARCHAR(50);`     |
| `AnalysisException: schema mismatch`                       | 스키마 필드명 불일치                        | Producer JSON 키와 StructField 이름 비교                                                |
| `checkpoint` 충돌 에러                                         | 이전 체크포인트 잔재                        | `docker exec train-spark-master rm -rf /tmp/checkpoint/*`                         |


## 포트 정리 — 헷갈리지 않는 법

```bash
맥북 ↔ PostgreSQL   : localhost:5433  (DataGrip, psql 등)
컨테이너 ↔ PostgreSQL: postgres:5432   (Spark JDBC, 컨테이너 내부)

맥북 ↔ Kafka        : localhost:9092  (로컬 Producer 테스트)
컨테이너 ↔ Kafka     : kafka:9092      (Spark readStream, 컨테이너 내부)
```

## 오타 토픽 확인 / 삭제


```bash
# 토픽 목록 확인
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list

# 오타 토픽에 데이터 있는지 확인
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic train-scedule \
  --from-beginning --max-messages 3

# 오타 토픽 삭제
docker exec -it train-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete --topic train-scedule
```

## VARCHAR 길이 초과 수정

```sql
-- DataGrip 에서 실행 (localhost:5433)
ALTER TABLE train_schedule ALTER COLUMN trn_plan_dptre_dt TYPE VARCHAR(50);
ALTER TABLE train_schedule ALTER COLUMN trn_plan_arvl_dt  TYPE VARCHAR(50);
```

## checkpoint 초기화

```bash
# 토픽 데이터가 있는데 Spark 가 읽지 않을 때
docker exec -it train-spark-master rm -rf /tmp/checkpoint/

# 특정 토픽만
docker exec -it train-spark-master rm -rf /tmp/checkpoint/schedule
```

---

✅ 완료되면 → [[05_Superset_Dashboard]] 으로 이동