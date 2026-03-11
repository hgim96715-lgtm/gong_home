---
aliases:
  - Spark Kafka Integration
  - readStream kafka
  - writeStream kafka
tags:
  - Streaming
  - Kafka
  - Spark
related:
  - "[[DataFrame_Aggregation]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_DataFrame]]"
  - "[[Spark_Functions_Library]]"
  - "[[Spark_Data_Cleaning]]"
  - "[[Spark_Streaming_JSON_ETL_Project]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
---

# Spark Streaming — Kafka 데이터 읽기 / 쓰기

## 한 줄 요약

> **Kafka 에서 데이터를 읽어(readStream) → 가공하고 → DB 나 Kafka 로 쓴다(writeStream).**

---

---

# ① 전체 구조

```
[Kafka Topic] ──readStream──▶ [Spark] ──가공──▶ [writeStream]──▶ [PostgreSQL / Kafka]
                                  │
                          value = Binary
                          → CAST AS STRING
                          → from_json (JSON 파싱)
```

```
Kafka 에서 읽어온 데이터는 항상 Binary 다.
사람이 읽을 수 있는 형태로 쓰려면 2단계 변환이 필요하다.

  1단계: Binary  → String   (CAST)
  2단계: String  → 컬럼들   (from_json + 스키마)
```

---

---

# ② 필수 패키지

```
스파크 기본에는 Kafka 연결 기능이 없다.
외부 패키지를 반드시 로드해야 한다.

spark-sql-kafka-0-10_2.12:3.5.0
                       ^^^^ Scala 버전
                              ^^^^^ Spark 버전 (자신의 Spark 버전과 맞춰야 함)
```

```bash
# spark-submit 실행 시 --packages 옵션으로 로드
spark-submit \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 \
  consumer.py
```

```python
# SparkSession 안에서 로드하는 방법 (Jupyter / 테스트용)
spark = (
    SparkSession.builder
    .appName("KafkaTest")
    .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0")
    .getOrCreate()
)
```

```
⚠️ 버전 불일치 시 에러 발생
   spark-sql-kafka-0-10_2.12:3.5.0  ← Spark 3.5.x 용
   spark-sql-kafka-0-10_2.12:3.4.0  ← Spark 3.4.x 용
   자신의 Spark 버전 확인: spark.version
```

---

---

# ③ Kafka 메시지 구조 — value 가 핵심

Kafka 에서 읽어오면 아래 고정 컬럼들이 자동으로 생긴다.

|컬럼|타입|설명|
|---|---|---|
|**value**|**Binary**|**실제 데이터 (가장 중요)**|
|key|Binary|메시지 키 (없을 수도 있음)|
|topic|String|토픽 이름|
|partition|Integer|파티션 번호|
|offset|Long|오프셋 번호|
|timestamp|Timestamp|메시지 생성 시간|

```
value 가 Binary 인 이유:
  Kafka 는 데이터 타입을 모른다 (그냥 바이트 덩어리로 저장)
  JSON 이든 Avro 든 String 이든 모두 Binary 로 들어옴
  → Spark 에서 직접 타입을 변환해줘야 함
```

---

---

# ④ readStream — Kafka 에서 읽기

```python
df = (
    spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")   # Kafka 브로커 주소
    .option("subscribe", "train-realtime")              # 구독할 토픽
    .option("startingOffsets", "latest")                # 시작 위치
    .load()
)
```

## 주요 옵션

```
kafka.bootstrap.servers
  Kafka 브로커 주소
  Docker 내부 통신이면 서비스명:포트 → "kafka:9092"
  외부(로컬 테스트)면 → "localhost:9092"

subscribe
  구독할 토픽 이름
  여러 개: "topic1,topic2,topic3"
  패턴 구독: subscribePattern 옵션으로 "topic.*"

startingOffsets
  "latest"    → 지금부터 들어오는 메시지만 읽음 (실시간)
  "earliest"  → 토픽에 쌓인 처음부터 전부 읽음 (재처리)

failOnDataLoss (기본값: true)
  Kafka 에서 오프셋이 유실됐을 때 Spark 를 어떻게 처리할지 결정하는 옵션

  "true"  → 오프셋 유실 감지 시 스트리밍 즉시 중단 (기본값)
  "false" → 오프셋 유실이 있어도 무시하고 계속 진행
```

## failOnDataLoss 언제 쓰나

```
오프셋 유실이 발생하는 상황:
  Kafka retention 이 짧아서 읽기 전에 메시지가 삭제됨
  Kafka 토픽을 삭제하고 재생성했을 때
  체크포인트에 저장된 오프셋이 Kafka 에 이미 없을 때
  개발/테스트 중 Kafka 를 재시작했을 때
```

```python
# 개발/테스트 환경 — 오프셋 유실 무시하고 계속 진행
df = (
    spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "train-realtime")
    .option("startingOffsets", "latest")
    .option("failOnDataLoss", "false")    # ← 추가
    .load()
)
```

```
⚠️ failOnDataLoss = false 주의사항:
  오프셋 유실 = 해당 구간 데이터가 영구적으로 누락됨
  false 로 설정하면 누락 사실을 Spark 가 무시하고 넘어감
  데이터 정합성이 중요한 운영 환경에서는 신중하게 사용

  false가 적합할때는 ? true가 적합할때는?
    실시간 전광판 데이터라 일부 누락이 허용됨 → false 가 적합
    지연 분석(train-delay) 처럼 정확성이 중요하면 → true 유지
```

---

---

# ⑤ Binary → 실제 데이터 변환

## 1단계 — Binary → String (CAST)

```python
from pyspark.sql.functions import col

# selectExpr 방식 (SQL 문법)
string_df = df.selectExpr("CAST(value AS STRING)")

# col 방식 (Python 방식)
string_df = df.select(col("value").cast("string"))
```

## 2단계 — String → 컬럼 분리 (from_json)

```python
from pyspark.sql.functions import from_json
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

# 수신할 JSON 의 스키마 정의 (Producer 가 보내는 구조와 일치해야 함)
schema = StructType([
    StructField("trn_no",       StringType()),
    StructField("mrnt_nm",      StringType()),
    StructField("dptre_stn_nm", StringType()),
    StructField("arvl_stn_nm",  StringType()),
    StructField("status",       StringType()),
    StructField("progress_pct", IntegerType()),
    StructField("data_type",    StringType()),
])

# JSON 문자열 → 컬럼들로 파싱
parsed_df = string_df.select(
    from_json(col("value"), schema).alias("data")
).select("data.*")   # 중첩 구조 펼치기
```

```
from_json 후 바로 select("data.*") 로 펼치는 이유:
  from_json 은 value 컬럼 하나를 StructType 컬럼(data) 으로 변환
  data.trn_no, data.mrnt_nm ... 처럼 중첩된 상태
  select("data.*") 로 펼치면 trn_no, mrnt_nm ... 각각 독립 컬럼이 됨
```

## 전체 파이프라인

```python
result_df = (
    df
    .selectExpr("CAST(value AS STRING) as value")     # 1. Binary → String
    .select(from_json(col("value"), schema).alias("data"))  # 2. String → Struct
    .select("data.*")                                  # 3. Struct 펼치기
)
```

---

---

# ⑥ writeStream — 결과 쓰기

## PostgreSQL 에 쓰기 (foreachBatch)

```
Streaming DataFrame 은 .write.jdbc() 를 직접 쓸 수 없다.
Streaming 은 데이터가 계속 흘러들어오는 상태라서
"지금 전부 저장해" 라는 명령을 내릴 수가 없기 때문.

foreachBatch 는 이 문제를 해결하는 방법:
  Spark 가 trigger 주기마다 데이터를 잘라서 배치(묶음)로 만들어줌
  그 배치를 일반 DataFrame 처럼 다룰 수 있게 함수로 넘겨줌
  → 함수 안에서 .write.jdbc() 사용 가능
```

```python
def write_to_postgres(batch_df, batch_id):
    if batch_df.isEmpty():
        return
    batch_df.write.jdbc(
        url="jdbc:postgresql://postgres:5432/train_db",
        table="train_realtime",
        mode="append",
        properties={
            "user":     "train_user",
            "password": "train_password",
            "driver":   "org.postgresql.Driver",
        },
    )

query = (
    result_df
    .writeStream
    .foreachBatch(write_to_postgres)
    .option("checkpointLocation", "/tmp/checkpoint/realtime")
    .trigger(processingTime="10 seconds")
    .start()
)
```

## batch_df / batch_id — Spark 가 자동으로 넘겨주는 인자

```
foreachBatch 에 넘기는 함수는 반드시 인자 2개를 받아야 한다.
Spark 가 자동으로 채워서 호출하기 때문에 이름은 바꿔도 되지만
순서와 개수는 고정이다.

def write_to_postgres(batch_df, batch_id):
                      ^^^^^^^^^  ^^^^^^^^
                      1번 인자   2번 인자

batch_df
  trigger 주기마다 잘린 데이터 묶음 — 일반 DataFrame 과 완전히 동일
  .show() / .count() / .write.jdbc() 등 모든 DataFrame API 사용 가능
  이 배치 안에 데이터가 없을 수도 있어서 isEmpty() 체크가 필요

batch_id
  배치에 Spark 가 부여하는 고유 번호 (0, 1, 2, 3 ...)
  trigger 마다 1씩 증가
  중복 저장 방지 / 디버깅 / 멱등성 처리에 활용
```

```python
# batch_id 활용 예시 — 로그 출력
def write_to_postgres(batch_df, batch_id):
    print(f"[batch {batch_id}] 처리 건수: {batch_df.count()}")
    if batch_df.isEmpty():
        return
    batch_df.write.jdbc(...)
```

## mode — 저장 방식

```
Streaming 에서 foreachBatch 안의 .write 는
일반 DataFrame 저장과 동일하게 mode 를 지정한다.
```

|mode|동작|언제 씀|
|---|---|---|
|`"append"`|기존 데이터 유지 + 새 데이터 추가|스트리밍 적재 (대부분 이 모드)|
|`"overwrite"`|테이블 전체를 지우고 새로 씀|배치 처리로 전체 재적재할 때|
|`"ignore"`|테이블이 이미 있으면 아무것도 안 함|초기 세팅용|
|`"error"`|테이블이 이미 있으면 에러 (기본값)|실수 방지용|

```
Streaming 에서는 거의 항상 "append"
  매 배치마다 새 데이터가 쌓여야 하기 때문
  "overwrite" 는 배치마다 테이블을 날려버리므로 스트리밍에서 사용 금지
```

---

## Kafka 로 다시 쓰기 (to_json)

````python
from pyspark.sql.functions import to_json, struct

# 여러 컬럼 → JSON 문자열 한 개로 (Kafka 는 value 컬럼 하나만 받음)
kafka_out_df = result_df.select(
    to_json(struct("*")).alias("value")
)

query = (
    kafka_out_df
    .writeStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("topic", "train-output")
    .option("checkpointLocation", "/tmp/checkpoint/output")
    .start()
)
````

```
to_json(struct("*")) 이 필요한 이유:
  Kafka 는 value 컬럼 하나만 받음
  컬럼이 여러 개일 때 → struct("*") 로 하나의 구조체로 묶고
                       → to_json() 으로 JSON 문자열로 변환
                       → .alias("value") 로 컬럼명 지정

  변환 예시:
    [trn_no="KTX001", status="운행중"] (컬럼 2개)
    → struct: {"trn_no": "KTX001", "status": "운행중"} (1개)
    → to_json: '{"trn_no": "KTX001", "status": "운행중"}' (문자열)
```

## 콘솔 출력 (테스트용)

```python
query = (
    result_df
    .writeStream
    .format("console")
    .option("truncate", False)
    .trigger(processingTime="5 seconds")
    .start()
)
```

---

---

# ⑦ parsed → final — DB 컬럼과 맞추기

```
from_json + select("data.*") 로 컬럼을 펼치면
JSON 에 있는 모든 필드가 컬럼으로 들어온다.

문제:
  JSON 에는 있지만 DB 테이블에는 없는 컬럼이 있을 수 있음
  순서도 DB 컬럼 순서와 다를 수 있음
  → write.jdbc 할 때 컬럼명/순서가 DB 와 정확히 맞지 않으면 에러 또는 잘못 저장

해결:
  final = parsed.select("컬럼1", "컬럼2", ...) 로
  DB 테이블 컬럼과 1:1 로 맞춰서 정리
```

```python
# parsed 상태 — JSON 에 있는 모든 필드 + created_at
# trn_no / mrnt_nm / dptre_stn_nm / arvl_stn_nm /
# plan_dep / plan_arr / status / progress_pct / data_type / created_at
# (DB 에 없는 컬럼이 섞여 있을 수 있음)

parsed = (
    raw
    .select(from_json(col("value").cast("string"), SCHEMA_REALTIME).alias("data"))
    .select("data.*")
    .withColumn("created_at", current_timestamp())
)

# final — DB train_realtime 테이블 컬럼과 정확히 일치하도록 선택
final = parsed.select(
    "trn_no", "mrnt_nm",
    "dptre_stn_nm", "arvl_stn_nm",
    "plan_dep", "plan_arr",
    "status", "progress_pct",
    "data_type", "created_at",
    # ⚠️ id 컬럼은 넣지 않음 — 아래 설명 참고
)
```

```text
⚠️ id 를 final 에 넣지 않는 이유

DB 테이블에서 id 는 SERIAL PRIMARY KEY 로 선언되어 있다.

CREATE TABLE train_realtime ( id SERIAL PRIMARY KEY, ← PostgreSQL 이 자동으로 1, 2, 3 ... 부여 trn_no VARCHAR(20), ... )

SERIAL 의 동작: INSERT 할 때 id 를 명시하지 않으면 DB 가 알아서 다음 번호를 채워줌 별도로 관리할 필요 없음

만약 Spark 에서 id 를 직접 넣으면: DB 의 자동 증가 시스템(시퀀스)과 충돌 중복 id 에러 또는 시퀀스 번호 체계가 꼬임

결론: id 는 final.select() 에서 제외 → write.jdbc 로 INSERT 시 id 컬럼을 생략 → PostgreSQL 이 자동으로 번호를 채워줌
```

```
final 이 없으면 생길 수 있는 문제:
  DB 에 없는 컬럼 포함 시 → write.jdbc 에서 column not found 에러
  컬럼 순서가 다를 시    → 엉뚱한 컬럼에 데이터가 들어갈 수 있음

final 은 "DB 에 넣기 직전 최종 정리 단계" 라고 보면 된다.
```

---

---

# ⑧ 클래스로 공통화 — _write_jdbc 클로저 패턴

```
토픽이 3개이면 foreachBatch 함수도 3개 만들어야 한다.
테이블 이름만 다르고 나머지 로직이 동일하면 중복 코드가 생긴다.

해결:
  테이블 이름을 인자로 받아서 write_batch 함수를 반환하는 메서드로 공통화
  → 클로저(Closure) 패턴
```

## 클로저란

```
함수가 함수를 반환하는 패턴.
바깥 함수의 변수(table)를 안쪽 함수(write_batch)가 기억한다.

_write_jdbc("train_schedule") 를 호출하면
  → table = "train_schedule" 을 기억하는 write_batch 함수가 반환됨
  → 이 write_batch 를 foreachBatch 에 넘기면 됨
```

```python
class TrainConsumer:

    def _write_jdbc(self, table: str):
        """테이블 이름을 받아서 foreachBatch 에 넘길 함수를 반환"""

        def write_batch(batch_df, batch_id):         # Spark 가 호출하는 함수
            if batch_df.isEmpty():
                return
            batch_df.write.jdbc(
                url=JDBC_URL,
                table=table,                         # 바깥의 table 변수를 기억
                mode="append",
                properties=JDBC_PROPS,
            )
            print(f"[{table}] batch_id={batch_id} | {batch_df.count()}건 저장")

        return write_batch                           # 함수 자체를 반환
```

```
호출 흐름:

  self._write_jdbc("train_schedule")
  ↓
  table = "train_schedule" 이 세팅된 write_batch 함수 반환
  ↓
  .foreachBatch(반환된_write_batch)
  ↓
  Spark 가 배치마다 write_batch(batch_df, batch_id) 자동 호출
```

## 실제 사용

```python
# ❌ 공통화 전 — 토픽마다 함수를 따로 만들어야 함
def write_schedule(batch_df, batch_id):
    batch_df.write.jdbc(..., table="train_schedule", ...)

def write_realtime(batch_df, batch_id):
    batch_df.write.jdbc(..., table="train_realtime", ...)  # 같은 코드 반복

def write_delay(batch_df, batch_id):
    batch_df.write.jdbc(..., table="train_delay", ...)     # 같은 코드 반복


# ✅ 공통화 후 — 테이블 이름만 바꿔서 재사용
final.writeStream.foreachBatch(self._write_jdbc("train_schedule"))
final.writeStream.foreachBatch(self._write_jdbc("train_realtime"))
final.writeStream.foreachBatch(self._write_jdbc("train_delay"))
```

## 전체 흐름 (run_schedule 기준)

```python
def run_schedule(self):
    # 1. Kafka 에서 읽기
    raw = self._read_stream("train-schedule")

    # 2. Binary → JSON 파싱 + created_at 추가
    parsed = (
        raw
        .select(from_json(col("value").cast("string"), SCHEMA_SCHEDULE).alias("data"))
        .select("data.*")
        .withColumn("created_at", current_timestamp())
    )

    # 3. DB 컬럼과 1:1 맞추기
    final = parsed.select(
        "run_ymd", "trn_no",
        "dptre_stn_cd", "dptre_stn_nm",
        "arvl_stn_cd",  "arvl_stn_nm",
        "trn_plan_dptre_dt", "trn_plan_arvl_dt",
        "data_type", "created_at",
    )

    # 4. PostgreSQL 에 쓰기
    return (
        final.writeStream
        .foreachBatch(self._write_jdbc("train_schedule"))  # 클로저로 테이블 지정
        .option("checkpointLocation", "/tmp/checkpoint/schedule")
        .trigger(processingTime="10 seconds")
        .start()
    )
```

```
raw    → Kafka 에서 읽은 원본 (value = Binary)
parsed → JSON 파싱 완료 + created_at 추가 (컬럼이 많거나 순서 불일치 가능)
final  → DB 컬럼과 정확히 일치하도록 정리 (write 직전)
```

## trigger — 배치 처리 주기

```python
.trigger(processingTime="10 seconds")   # 10초마다 처리
.trigger(processingTime="1 minute")     # 1분마다 처리
.trigger(once=True)                     # 한 번만 실행 (배치처럼)
```

## checkpointLocation — 장애 복구

```python
.option("checkpointLocation", "/tmp/checkpoint/토픽명")
```

```
체크포인트란:
  "나 여기까지 읽었어" 를 파일로 기록해두는 것
  Spark 가 죽었다 살아나도 이어서 읽을 수 있음 (Exactly-once)
  토픽마다 별도 경로로 관리하는 것이 원칙
```

## awaitTermination — 스트림 유지

```python
query.awaitTermination()           # 단일 쿼리
spark.streams.awaitAnyTermination() # 여러 쿼리 동시 실행 시
```

```
awaitTermination 없으면:
  start() 는 백그라운드 실행
  코드가 바로 끝나버려서 스트리밍이 즉시 종료됨
  → 반드시 마지막에 awaitTermination 호출
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`AnalysisException: Failed to find data source: kafka`|spark-sql-kafka 패키지 없음|`--packages` 옵션 또는 `.config("spark.jars.packages", ...)` 추가|
|`AnalysisException: Missing option 'subscribe'`|토픽 지정 안 함|`.option("subscribe", "토픽명")` 추가|
|`ModuleNotFoundError: No module named 'pyspark'`|로컬에서 실행함|Docker 컨테이너 안에서 실행 (`docker exec -it spark-master ...`)|
|`SparkException: Some data may have been lost`|오프셋 유실 (Kafka retention 만료 등)|`.option("failOnDataLoss", "false")` — 단, 데이터 누락 감수|
|value 가 null|스키마 불일치|Producer 가 보내는 JSON 키와 StructField 이름 일치 확인|
|결과가 안 뜸|체크포인트 오프셋 고착|checkpoint 폴더 삭제 후 재실행|
|패키지 다운로드 매번 느림|Maven 에서 반복 다운로드|.jar 미리 받아서 `--jars` 로 지정|

---

---

# 치트시트

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, to_json, struct
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

spark = SparkSession.builder.appName("KafkaApp").getOrCreate()

# 스키마 정의
schema = StructType([
    StructField("trn_no",  StringType()),
    StructField("status",  StringType()),
    StructField("delay",   IntegerType()),
])

# 읽기
df = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "train-realtime")
    .option("startingOffsets", "latest")
    .load()
)

# 변환
result = (
    df
    .selectExpr("CAST(value AS STRING) as value")
    .select(from_json(col("value"), schema).alias("d"))
    .select("d.*")
)

# DB 쓰기
def to_pg(batch_df, batch_id):
    if batch_df.isEmpty(): return
    batch_df.write.jdbc(
        url="jdbc:postgresql://postgres:5432/train_db",
        table="train_realtime",
        mode="append",
        properties={"user": "...", "password": "...", "driver": "org.postgresql.Driver"},
    )

query = (
    result.writeStream
    .foreachBatch(to_pg)
    .option("checkpointLocation", "/tmp/ckpt/realtime")
    .trigger(processingTime="10 seconds")
    .start()
)

spark.streams.awaitAnyTermination()
```