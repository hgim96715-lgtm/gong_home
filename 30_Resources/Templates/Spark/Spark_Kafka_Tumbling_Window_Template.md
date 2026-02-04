---
aliases:
  - Tumbling Window Template
  - Spark Window Aggregation
  - 스파크 윈도우 집계 템플릿
tags:
  - Spark
  - Streaming
  - Kafka
  - Snippet
description: Kafka에서 JSON 데이터를 읽어 시간 기준(Tumbling Window)으로 집계하는 템플릿
related:
  - "[[Spark_Streaming_Window_Aggregation]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Streaming_Architecture]]"
  - "[[Spark_Streaming_Watermark]]"
---

## 🚀 Spark Streaming: Tumbling Window Aggregation

카프카에서 들어오는 시계열(Time-series) 데이터를 **일정 시간 간격(예: 5분)**으로 잘라서 집계하는 코드입니다.

### 📋 코드 스니펫

```python
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, IntegerType
from pyspark.sql.functions import from_json, col, to_timestamp, window, sum

# 1. Spark Session 설정
spark = (
    SparkSession
    .builder
    .master("spark://spark-master:7077")  
    .appName("kafkaTumblingTime")              
    .config("spark.streaming.stopGracefullyOnShutdown", "true")
    .config("spark.sql.shuffle.partitions", "3") 
    .getOrCreate()
)

# 2. JSON 데이터 스키마 정의
schema = StructType([
    StructField("create_date", StringType()),
    StructField("amount", IntegerType())
])

# 3. Kafka Source 연결
events = (
    spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka:9092")
    .option("subscribe", "timeseries") # 토픽 이름 확인       
    .option("startingOffsets", "earliest")
    .option("failOnDataLoss", "false")
    .load()
)

# 4. 데이터 파싱 (Binary -> String -> JSON)
value_df = events.select(
    col('key'),
    from_json(col("value").cast("string"), schema).alias("value")
)

# 5. 시간 타입 변환 (String -> Timestamp) 
# 주의: 포맷 대소문자 확인 (MM:월, mm:분 / HH:24시)
timestamp_format = "yyyy-MM-dd HH:mm:ss"

tf_df = (
    value_df
    .select("value.*")
    .withColumn("create_date", to_timestamp("create_date", timestamp_format))
)

# 6. Tumbling Window 집계 (겹치지 않음)
window_duration = "5 minutes"

window_agg_df = (
    tf_df
    .groupBy(window(col("create_date"), window_duration))                
    .agg(sum("amount").alias("total_amount"))
)

# 7. 결과 출력 (Update 모드)
query = (
    window_agg_df
    .writeStream
    .outputMode("update") # 집계 결과가 바뀔 때마다 출력 (Complete도 가능)
    .format("console")
    .option("truncate", "false") # 내용이 길어도 자르지 않음
    .trigger(processingTime="5 seconds") # 5초마다 마이크로 배치 실행 (부하 조절)
    .start()
)

query.awaitTermination()
```

