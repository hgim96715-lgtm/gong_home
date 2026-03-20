---
aliases:
  - Kafka
  - PostgreSQL 적재
  - Spark Streaming
  - er-realtime Consumer
  - Hospital Kafka Spark
tags:
  - Project
related:
  - "[[00_Hospital_Project]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
  - "[[Spark_JDBC_PostgreSQL]]"
  - "[[Spark_JSON_Handling]]"
  - "[[Spark_DataFrame_Transform]]"
  - "[[Spark_Session_Context]]"
  - "[[Spark_Transformations_vs_Actions]]"
  - "[[Spark_Functions_Library]]"
  - "[[Docker_Container_Interaction]]"
---
---
---

# 04_Hospital_Kafka_Spark — Kafka 토픽 생성 + Spark Consumer

## 이 단계의 목표

```
① Kafka 토픽 생성 (er-realtime / er-hospitals)
② CLI Consumer 로 Producer 메시지 확인
③ Spark Structured Streaming — er-realtime 토픽 → PostgreSQL 적재
④ 적재 확인 (DataGrip)
```

---

---

## 왜 Kafka Consumer 가 아니라 Spark Structured Streaming 인가?

```
특징:
  마이크로 배치 처리 — 30초마다 묶어서 처리
  분산 처리 가능 — 대량 데이터도 병렬 처리
  DataFrame API — 집계/변환/조인 SQL 처럼 가능
  장애 복구 — checkpointLocation 으로 재시작해도 안전

Hospital 프로젝트가 적합한 이유:
  병원 수: 417개 × 5분마다 = 하루 12만건 (대량)
  향후 집계 로직 추가 가능 (지역별 평균 병상 등)
  Spark 생태계 경험 — 포트폴리오 목적
```

## 비교 정리

|항목|KafkaConsumer|Spark Structured Streaming|
|---|---|---|
|처리 방식|건별 실시간|마이크로 배치 (N초마다)|
|처리량|소량 적합|대량 적합|
|변환/집계|직접 Python 코드|DataFrame API (SQL 수준)|
|장애 복구|직접 구현 필요|checkpoint 자동 관리|
|복잡도|단순|상대적으로 복잡|
|적합한 상황|단순 적재 / 소량|대량 / 집계 포함 / 확장 필요|

---

---

# ① Kafka 토픽 확인 / 추가 생성

```bash
# 현재 토픽 목록 확인
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --list
    
# er-realtime 토픽 생성
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic er-realtime --partitions 1 --replication-factor 1
    
# er-hospitals 토픽 생성 (Airflow 배치용)
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic er-hospitals --partitions 1 --replication-factor 1

# 토픽 상세 확인
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --describe \
    --topic er-realtime
```

---

---

# ② CLI Consumer — Producer 메시지 확인

```
Producer 를 실행한 상태에서
CLI Consumer 로 Kafka 에 메시지가 쌓이는지 확인
```

```bash
# er-realtime 토픽 실시간 확인
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic er-realtime \
    --from-beginning \
    --max-messages 3
```

```json
// 정상 출력 예시
{
  "hpid": "A2200008",
  "hpname": "강릉아산병원",
  "hvec": 1,
  "hvoc": 14,
  "hvctayn": "Y",
  "hvventiayn": "Y",
  "notice_msg": "소아응급환자 정상 수용 가능합니다.",
  "data_type": "er_realtime",
  "created_at": "2026-03-18T20:06:32.938259"
}
```

---

---

# ③ 폴더 구조

```
spark/
├── spark_consumer.py    ← Spark Streaming Consumer
└── requirements.txt
```

---

---

# ④ Spark Consumer 코드 — consumer.py

```python
import os
from dotenv import load_dotenv
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import (StructType, StructField, StringType, IntegerType)

load_dotenv()

# ── 설정 ────────────────────────────────────────────────
KAFKA_BOOTSTRAP  = os.getenv("KAFKA_BOOTSTRAP_SERVERS", "kafka:9092")
POSTGRES_DB      = os.getenv("POSTGRES_DB",       "hospital_db")
POSTGRES_USER    = os.getenv("POSTGRES_USER",     "hospital_user")
POSTGRES_PASSWORD = os.getenv("POSTGRES_PASSWORD", "hospital_password")

POSTGRES_HOST = "postgres"   # 컨테이너 내부 통신 → 서비스명 사용
POSTGRES_PORT = "5432"       # 컨테이너 내부 포트

POSTGRES_URL = f"jdbc:postgresql://{POSTGRES_HOST}:{POSTGRES_PORT}/{POSTGRES_DB}"

JDBC_PROPS = {
    "user":     POSTGRES_USER,
    "password": POSTGRES_PASSWORD,
    "driver":   "org.postgresql.Driver"
}

# ── er_realtime 스키마 ───────────────────────────────────
schema = StructType([
    StructField("hpid",       StringType()),
    StructField("hpname",     StringType()),
    StructField("hvec",       IntegerType()),   # 음수 가능 (설계 고민 노트 참고)
    StructField("hvoc",       IntegerType()),
    StructField("hvgc",       IntegerType()),   # 일반 입원실
    StructField("hvctayn",    StringType()),    # Y/N (parse_yn 처리됨)
    StructField("hvventiayn", StringType()),    # Y/N (parse_yn 처리됨)
    StructField("hvmriayn",   StringType()),    # MRI 가용 Y/N
    StructField("hvangioayn", StringType()),    # 조영촬영기 가용 Y/N
    StructField("notice_msg", StringType()),
    StructField("data_type",  StringType()),
    StructField("created_at", StringType()),
    # region / duty_addr → er_hospitals 에서 JOIN 으로 조회
])

# ── Spark 세션 ───────────────────────────────────────────
spark = (
    SparkSession
    .builder
    .master("spark://spark-master:7077")
    .appName("HospitalConsumer")
    .config("spark.sql.shuffle.partitions", "2")
    .getOrCreate()
)
spark.sparkContext.setLogLevel("WARN")

# ── Kafka 읽기 ───────────────────────────────────────────
df_raw = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", KAFKA_BOOTSTRAP)
    .option("subscribe", "er-realtime")
    .option("startingOffsets", "latest")
    .load()
)

# ── JSON 파싱 + created_at 타입 변환 ─────────────────────
df = (
    df_raw
    .select(col("value").cast("string").alias("json_str"))
    .select(from_json(col("json_str"), schema).alias("data"))
    .select("data.*")
    .withColumn("created_at", col("created_at").cast("timestamp"))
)

# ── PostgreSQL 적재 (count 변수에 캐싱 — Action 1번) ──────
def write_to_postgres(batch_df, batch_id):
    batch_count = batch_df.count()   # Action 1번으로 저장
    if batch_count == 0:
        return
    batch_df.write.jdbc(
        url=POSTGRES_URL,
        table="er_realtime",
        mode="append",
        properties=JDBC_PROPS
    )
    print(f"[배치 {batch_id}] {batch_count}건 → er_realtime DB 적재 완료!")

# ── 스트리밍 시작 ────────────────────────────────────────
query = (
    df
    .writeStream
    .foreachBatch(write_to_postgres)
    .option("checkpointLocation", "/tmp/hospital_checkpoint")
    .trigger(processingTime="30 seconds")
    .start()
)

print("🏥 Spark Consumer 시작 — er-realtime → PostgreSQL 적재 대기 중...")
query.awaitTermination()
```

---

---

# ⑤ 실행

>[[Docker_Container_Interaction#⑥ docker cp — 로컬 ↔ 컨테이너 파일 복사]] 참고 
>[[Kafka_CLI_Cheatsheet]] 토픽 생성 참고 

```bash
# 1. 컨테이너 실행
docker compose up -d
docker compose ps

# 2. Producer 실행 (STEP 3)
cd producer && python3 producer.py &

# 3. Consumer 최초 실행 시 — JAR + .env + python-dotenv 준비
#    (docker compose down 후 재시작 시도 매번 필요)

# JAR 복사 (hospital-spark-master = docker-compose.yml 의 container_name)
docker cp spark_drivers/. hospital-spark-master:/opt/spark/jars/

# .env 복사 (load_dotenv() 가 컨테이너 안에서 읽어야 함)
docker cp .env hospital-spark-master:/opt/spark/apps/.env

# python-dotenv 설치
docker exec -u root -it hospital-spark-master pip3 install python-dotenv

# 4. spark-submit 실행
docker exec -it hospital-spark-master \
  /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --jars /opt/spark/jars/spark-sql-kafka-0-10_2.12-3.5.0.jar,/opt/spark/jars/spark-token-provider-kafka-0-10_2.12-3.5.0.jar,/opt/spark/jars/kafka-clients-3.4.1.jar,/opt/spark/jars/commons-pool2-2.11.1.jar,/opt/spark/jars/postgresql-42.7.1.jar \
  /opt/spark/apps/consumer.py
```

```
정상 실행 시:
  🏥 Spark Consumer 시작 — er-realtime → PostgreSQL 적재 대기 중...
  [배치 1] 417건 → er_realtime DB 적재 완료!
  [배치 2] 417건 → er_realtime DB 적재 완료!
```

---

---

# ⑥ 적재 확인

>[[PostgreSQL_Setup#④ psql — PostgreSQL 전용 CLI 클라이언트]] 참고 

```bash
# psql 로 확인
docker exec -it hospital-postgres \
    psql -U hospital_user -d hospital_db \
    -c "SELECT COUNT(*) FROM er_realtime;"

docker exec -it hospital-postgres \
    psql -U hospital_user -d hospital_db \
    -c "SELECT hpid, hpname, hvec, hvctayn, notice_msg, created_at
        FROM er_realtime
        ORDER BY created_at DESC
        LIMIT 5;"
```

```
DataGrip 에서 확인:
  localhost:5434 → hospital_db → public → er_realtime
  SELECT * FROM er_realtime LIMIT 10;
```

| id  | hpid     | hpname          | hvec | hvoc | hvctayn | hvventiayn | notice_msg                  | data_type   | created_at                 |
| --- | -------- | --------------- | ---- | ---- | ------- | ---------- | --------------------------- | ----------- | -------------------------- |
| 1   | A2200008 | 강릉아산병원          | 15   | 15   | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |
| 2   | A2200001 | 연세대학교원주세브란스기독병원 | 23   | 15   | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |
| 3   | A2200013 | 한림대학교춘천성심병원     | 11   | 8    | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |
| 4   | A2100042 | 의료법인명지의료재단명지병원  | 10   | 13   | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |
| 5   | A2100033 | 국민건강보험공단일산병원    | 20   | 2    | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |
| 6   | A2100005 | 순천향대학교부속부천병원    | 9    | 21   | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |
| 7   | A2100001 | 분당서울대학교병원       | -9   | 38   | Y       | Y          | 119 구급대원께서는 문의 후 이송 부탁드립니다. | er_realtime | 2026-03-19 22:18:00.969869 |


---

---

# 설계 고민 노트

```
trigger(processingTime="30 seconds"):
  30초마다 배치 처리
  Producer 는 5분마다 발행 → 30초 배치면 충분히 빠름
  너무 짧으면 PostgreSQL 부하 증가

startingOffsets="latest":
  Consumer 시작 시점부터 새로운 메시지만 처리
  "earliest" 로 바꾸면 토픽 처음부터 전부 처리

checkpointLocation:
  Spark 가 어디까지 읽었는지 저장
  Consumer 재시작해도 중복 없이 이어서 처리

hvec 음수 처리 고민:
  실제 데이터에서 hvec = -9 같은 음수 값이 나옴
  API 에서 음수 = 병상 초과 / 대기 환자 있음 으로 해석 가능
  Superset / SQL 에서 처리할 방향:
    hvec > 0  → 수용 가능
    hvec <= 0 → 포화 (병상 없음 / 초과)
  → 음수도 그대로 DB 에 저장 (INT 타입) / 필터링은 쿼리에서 처리

notice_msg 중복 문제:
  현재 모든 병원의 notice_msg 가 동일한 문구로 나옴
  "119 구급대원께서는 문의 후 이송 부탁드립니다."
  원인 추정:
    ① 병원들이 시스템 기본값 메시지 그대로 사용
    ② 메시지 API 가 실제로 병원별 맞춤 메시지를 잘 안 보냄
  향후 처리 방향:
    중복 메시지는 실질적으로 의미 없음
    Superset 에서 notice_msg != '119 구급대원...' 필터로 의미있는 메시지만 표시
    또는 notice_msg IS NOT NULL AND notice_msg != '' 로 유효 메시지만 카운팅
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`ClassNotFoundException: org.postgresql.Driver`|PostgreSQL JAR 없음|`--packages` 에 `org.postgresql:postgresql:42.6.0` 추가|
|`KafkaException: Failed to construct kafka consumer`|Kafka 연결 실패|`KAFKA_BOOTSTRAP_SERVERS` 확인 (컨테이너 내부: `kafka:9092`)|
|`er_realtime` 에 데이터 안 쌓임|Producer 미실행|Producer 먼저 실행 후 Consumer 시작|
|`AnalysisException: cannot resolve`|스키마 컬럼명 불일치|`schema` 정의와 `er_realtime` 테이블 컬럼 확인|
|checkpoint 에러|이전 체크포인트 충돌|`rm -rf /tmp/hospital_checkpoint` 후 재시작|

✅ 완료되면 → [[05_Hospital_Superset]] 으로 이동