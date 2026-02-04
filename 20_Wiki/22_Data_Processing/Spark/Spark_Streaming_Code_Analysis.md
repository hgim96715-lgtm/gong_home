---
aliases:
  - Code Analysis
  - 코드 분석
  - Kafka Cassandra Join 분석
tags:
  - Spark
  - Code
related:
  - "[[Spark_Streaming_Cassandra_Setup]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/kafka_static_join.py
---
##  코드 한 줄 한 줄 뜯어보기 (Line-by-Line Guide)

**`kafka_static_join.py`** 코드가 도대체 무슨 일을 하는지, 요리 레시피 보듯이 단계별로 설명합니다.

---
## 1단계: 재료 준비 (설정 & 라이브러리) 

```python
from pyspark.sql import SparkSession
# ... (라이브러리 import 생략)

spark = (
    SparkSession
    .builder
    # 1. 마스터 노드 주소: 누가 대장인지 알려줍니다.
    .master("spark://spark-master:7077")  
    .appName("staticJoin")              
    
    # 2. 종료 설정: 강제 종료 대신 하던 일은 다 끝내고 얌전히 꺼지게 합니다. (데이터 보호)
    .config("spark.streaming.stopGracefullyOnShutdown", "true")
    
    # 3. 셔플 파티션: 데이터 섞을 때 몇 조각으로 나눌지. (기본값 200은 너무 커서 3으로 줄임)
    .config("spark.sql.shuffle.partitions", "3")
    
    # 4. 카산드라 연결 정보 (중요!)
    .config("spark.cassandra.connection.host", "cassandra") # Docker 컨테이너 이름
    .config("spark.cassandra.connection.port", "9042")      # 통신 포트
    .config("spark.cassandra.auth.username", "cassandra")   # 아이디
    .config("spark.cassandra.auth.password", "cassandra")   # 비번
    
    # 5. 카산드라 확장 기능: 스파크가 카산드라를 '테이블'처럼 쓰기 위해 필요한 도구 상자
    .config("spark.sql.extensions", "com.datastax.spark.connector.CassandraSparkExtensions")
    .config("spark.sql.catalog.lh", "com.datastax.spark.connector.datasource.CassandraCatalog")
    .getOrCreate()
)
```

**💡 핵심:** 카산드라랑 대화하려면 `connection.host`와 `auth`(아이디/비번) 정보가 필수입니다. 이 부분이 틀리면 연결 자체가 안 됩니다.

- `docker-compose.yml`에는 별도의 비밀번호 설정 환경변수(예: `CASSANDRA_PASSWORD`)가 없다면 
- 기본값 카산드라(Cassandra)의 **기본(Default) 아이디와 비밀번호**는 모두 **`cassandra`** 이다 
- 아니면 , 확인차 시도해볼수 있다 
```bash
# 컨테이너 내부에서 로그인 시도 (-u 아이디 -p 비밀번호)
docker exec -it cassandra cqlsh -u cassandra -p cassandra
```

- **성공 시:** `cqlsh>` 프롬프트가 뜨면서 접속됩니다.
- **실패 시:** `AuthenticationFailed` 에러가 뜹니다.

## 2단계: 카프카에서 재료 꺼내오기 (Source)

```python
# 1. 스키마 정의: "데이터가 이렇게 생겼을 거야"라고 미리 틀을 잡습니다.
schema = StructType([
    StructField("create_date", StringType()),
    StructField("login_id", StringType())
])

events = (
    spark
    .readStream             # "나 지금 스트리밍 읽을 거야!"
    .format("kafka")        # 어디서? 카프카에서.
    .option("kafka.bootstrap.servers", "kafka:9092") # 카프카 주소
    .option("subscribe", "casan")      # 구독할 토픽 이름
    .option("startingOffsets", "earliest") # "처음부터 싹 다 읽어와" (latest는 지금부터)
    .load()
)
```

---
## 3단계: 재료 손질하기 (Parsing & Transform)

카프카는 데이터를 **바이트(이진 데이터)** 덩어리로 줍니다. 이걸 우리가 읽을 수 있게 바꿔야 합니다.

```python
value_df = events.select(
    col('key'),
    # 1. cast("string"): 바이트 덩어리를 문자열(String)로 변환
    # 2. from_json: 문자열을 아까 만든 스키마(틀)에 맞춰서 분해
    from_json(col("value").cast("string"), schema).alias("value")
)

timestamp_format = "yyyy-MM-dd HH:mm:ss"

event_df = (
    value_df
    .select("value.*") # value 안에 있는 create_date, login_id를 밖으로 꺼냄
    # 문자열 날짜("2023-09-01...")를 스파크가 계산할 수 있는 '시간' 타입으로 변환
    .withColumn("create_date", to_timestamp("create_date", timestamp_format))
)
```

---
## 4단계: 냉장고에서 반찬 꺼내기 (Static Data Read)

실시간 데이터엔 `login_id`만 있고 이름이 없죠? 카산드라(냉장고)에서 유저 정보를 가져옵니다.

```python
user_df = (
    spark
    .read # readStream이 아니라 그냥 read입니다. (정적 데이터)
    .format("org.apache.spark.sql.cassandra") # 카산드라 포맷 사용
    .option("keyspace", "test_db") # 데이터베이스 이름
    .option("table", "users")      # 테이블 이름
    .load()
)
```

- 테이블은 이미 만들어져 있어야 한다! 

----
## 5단계: 섞어서 요리하기 (Join)

실시간 데이터(`event_df`)와 저장된 데이터(`user_df`)를 합칩니다.

```python
join_df = event_df.join(
    user_df,
    event_df.login_id == user_df.login_id, # 아이디가 같은 사람끼리 묶어라
    "inner" # 교집합: 카산드라에 정보가 있는 사람만 남김 (없는 사람은 버림)
).drop(user_df.login_id) # 아이디 컬럼이 두 개 생기니까 하나는 버림
```

---
## 6단계: 그릇에 담기 (Output Select) 

카산드라 테이블 모양에 딱 맞춰서 데이터를 정리해줍니다.

```python
output_df = join_df.select(
    col("login_id"),                        # Cassandra의 PK 이름과 일치
    col("user_name"),                       # Cassandra 컬럼명과 일치
    col("create_date").alias("last_login")  # Spark에선 create_date지만, Cassandra엔 last_login이라서 이름표 교체! )
)
```

- 카산드라 테이블에는 없는데 스파크 데이터프레임에 컬럼이 남아있으면, 커넥터가 "이거 어디다 넣어요?" 하고 에러를 뱉습니다.
- **"컬럼 이름(Column Name)"과 "데이터 타입(Type)"은 무조건 일치해야 한다!!**
- 카산드라에 넣기 직전에는 **반드시 `.select()`를 써서 카산드라 테이블 컬럼과 이름/개수를 똑같이 맞춰주는 게 좋다**

---
## 7단계: 서빙하기 (Write Stream) 

여기가 제일 중요합니다! 스트리밍 데이터를 외부 DB(카산드라)에 저장하는 부분입니다.

```python
# 배치(Batch): 스파크는 스트리밍을 작은 덩어리(Batch)로 쪼개서 처리합니다.
# 함수 안으로 들어오면 "정적 데이터(Batch)"가 됩니다. (write)
def cassandraWriter(batch_df, batch_id):
	# 여기 들어온 batch_df는 더 이상 스트리밍이 아닙니다!
	# 그냥 멈춰있는 일반 데이터프레임입니다.
    # 이 함수는 5초마다 한 번씩 실행됩니다.
    batch_df.write \ # writeStream이 아니라 그냥 write! (일반 저장)
        .format("org.apache.spark.sql.cassandra") \
        .option("keyspace", "test_db") \
        .option("table", "users") \
        .mode("append") \
        .save() # "저장 땅땅!"
    # .mode("append")지만 카산드라 특성상 PK가 같으면 덮어쓰기(Update)가 됩니다!

# 밖에서는 "스트리밍"입니다. (writeStream)
query = (
    output_df
    .writeStream  # "나는 계속 흐르는 중이야~"
    .foreachBatch(cassandraWriter) # "근데 저장할 때는 저 함수 불러서 처리해!"
    .outputMode("update")          # 결과 갱신 모드
    .trigger(processingTime="5 seconds") # 5초마다 실행
    .start()
)
```

### `foreachBatch`가 뭔가요?

- "배치(Batch) **마다(foreach)** 이 함수를 실행해라."
- **역할:** 스트리밍 데이터는 **끝없이 흐르는 물**과 같습니다. 이걸 DB에 저장하려면 **"물을 바가지(Batch)로 퍼서"** 옮겨 담아야 합니다.
- **동작**
	- 5초 동안 들어온 데이터를 모아서 **작은 데이터프레임(`batch_df`)** 으로 만듭니다.
	- 우리가 정의한 함수(`cassandraWriter`)에게 이 **작은 데이터프레임**을 넘겨줍니다.
	- 함수 안에서 **"저장해(`save`)"** 명령을 실행합니다.

### 왜 이렇게 복잡하게 할까?

그냥 `.format("cassandra").start()` 하면 안 되나요? -> **되긴 하지만, `foreachBatch`를 쓰는 이유가 있습니다.**

- **다중 작업 가능:**
	- 데이터가 들어오면 **Cassandra에도 저장**하고, 동시에 **Console에도 찍어보고**, **SQL에도 넣고** 싶을 수 있죠?
	- `foreachBatch` 함수 안에는 코드를 여러 줄 쓸 수 있으니, 한 번 읽어서 여러 곳에 저장이 가능합니다.
- **기능 지원:**
	- 일부 데이터베이스 커넥터는 `writeStream`을 직접 지원하지 않고, 일반 `write`만 지원하는 경우가 있습니다.
	- 이때 `foreachBatch`가 **통역사 역할**을 해줍니다. (스트리밍 -> 일반 배치로 변환)

**한번 읽어서 여러곳 저장?**

```python
def multiSinkWriter(batch_df, batch_id):
    # [중요] 데이터를 여러 번 쓸 거라면 캐시(Cache)를 하는 게 좋습니다.
    # 안 그러면 Cassandra 저장할 때 한 번 계산하고, 화면에 찍을 때 또 계산합니다. (비효율!)
    batch_df.persist() 

    print(f"--- Batch ID: {batch_id} 시작 ---")

    # 1️⃣ 메인 저장: Cassandra에 Upsert
    batch_df.write \
        .format("org.apache.spark.sql.cassandra") \
        .option("keyspace", "test_db") \
        .option("table", "users") \
        .mode("append") \
        .save()
    print(">> Cassandra 저장 완료")

    # 2️⃣ 모니터링: 화면에 상위 5개만 출력
    batch_df.show(5, False)

    # 3️⃣ 분기 처리: 'Park'씨 데이터만 따로 JSON 파일로 백업 (가정)
    # 일반 데이터프레임처럼 filter 사용 가능!
    vip_user_df = batch_df.filter("user_name == 'Park'")
    
    if vip_user_df.count() > 0:
        vip_user_df.write \
            .mode("append") \
            .json("./data/vip_users_backup")
        print(">> VIP 유저 백업 완료")

    # [중요] 다 썼으면 메모리에서 비워줍니다.
    batch_df.unpersist()
```

