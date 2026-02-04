---
aliases:
  - Spark Kafka Template
  - 스파크 카프카 템플릿
  - Stateful Snippet
tags:
  - Spark
  - Streaming
  - Kafka
  - Snippet
description: Kafka에서 데이터를 읽어 GroupBy 집계(Stateful)를 수행하는 기본 템플릿
related:
  - "[[00_Apache_Spark_HomePage]]"
---
## Spark Structured Streaming (Kafka + Stateful Aggregation)

이 코드는 **"카프카에서 데이터를 읽어서 -> 집계(Sum/Count)하고 -> 콘솔에 누적 결과(Complete)를 출력"** 하는 패턴입니다.

### 📋 코드 스니펫

```python
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, IntegerType
from pyspark.sql.functions import from_json, col, sum

# 1. Spark Session 생성 (설정 확인 필수)
spark = (
    SparkSession
    .builder
    .master("spark://spark-master:7077")  # 로컬 테스트 시 "local[3]"
    .appName("kafkaStateful")             # 앱 이름 변경
    .config("spark.streaming.stopGracefullyOnShutdown", "true")
    .config("spark.sql.shuffle.partitions", "3") # 클러스터 코어 수에 맞춰 조정
    .getOrCreate()
)

# 2. 스키마 정의 (데이터 구조에 맞게 수정)
schema = StructType([
    StructField("product_id", StringType()),
    StructField("amount", IntegerType())
])

# 3. Kafka Source 연결
events = (
    spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "pos")           # 토픽 이름 변경
    .option("startingOffsets", "earliest")
    .option("failOnDataLoss", "false")
    .load()
)

# 4. JSON 파싱 (Value 컬럼 추출)
value_df = events.select(
    col('key'),
    from_json(col("value").cast("string"), schema).alias("value")
)

# 5. 데이터 평탄화 (Flatten)
tf_df = value_df.selectExpr(
    'value.product_id',
    'value.amount'
)

# 6. Stateful 집계 로직 (GroupBy)
total_df = (
    tf_df
    .groupBy("product_id")                # 그룹핑 기준
    .agg(sum("amount").alias("total_amount")) # 집계 함수
)

# 7. Sink 출력 (Console + Complete 모드)
# 주의: 집계(Agg)가 있으면 'append' 모드는 불가능함 (Watermark 없을 시)
query = (
    total_df
    .writeStream
    .outputMode("complete") # 전체 결과를 덮어쓰기 (Stateful 필수)
    .format("console")
    .option("checkpointLocation", "chk/stateful_query") # 체크포인트 필수
    .start()
)

query.awaitTermination()
```


