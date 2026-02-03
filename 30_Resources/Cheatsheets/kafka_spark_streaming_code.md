


```python
from pyspark.sql import SparkSession

import time

  

# 1. 스파크 세션 생성

# 주의: .py로 실행할 때는 여기서 패키지 설정을 하거나, spark-submit 명령어에 옵션을 줘야 합니다.

# 코드에 포함하는 것이 관리하기 편합니다.

spark = SparkSession.builder \

.appName("KafkaPyTest") \

.config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1") \

.getOrCreate()

  

spark.sparkContext.setLogLevel("WARN") # 로그가 너무 많이 뜨는 것 방지

  

print("🚀 스파크 애플리케이션이 시작되었습니다!")

  

# 2. 카프카 읽기 (Read)

df = spark \

.readStream \

.format("kafka") \

.option("kafka.bootstrap.servers", "kafka:9092") \

.option("subscribe", "test-topic") \

.option("startingOffsets", "earliest") \

.load()

  

# 3. 데이터 변환 (Binary -> String)

df = df.selectExpr("CAST(value AS STRING)")

  

# 4. 콘솔에 출력 (Write)

print("👉 데이터 스트리밍을 시작합니다... (Ctrl+C로 종료하세요)")

query = df \

.writeStream \

.format("console") \

.outputMode("append") \

.start()

  

query.awaitTermination()
```
