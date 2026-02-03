---
aliases:
  - Multiple Sinks
  - awaitAnyTermination
  - 데이터 분기
tags:
  - Spark
  - Streaming
  - Kafka
related:
  - "[[Spark_Streaming_JSON_ETL_Project]]"
  - "[[Spark_JSON_Handling]]"
---
##  Sink to Multiple Places (다중 출력) 

스트리밍 데이터는 한 번 흘러가면 끝입니다. 
하지만 우리는 가공한 데이터를 **콘솔에서도 보고(Console)** 동시에 **카프카 토픽에도 저장(Kafka)** 하고 싶을 때가 있습니다.
 **분석용 파일 저장**과 **실시간 전송**을 동시에 처리할 때 사용합니다. 

### ⚠️ 주의사항: `.start()`는 두 번 쓸 수 없다?

일반적으로 한 쿼리에 `.start()`를 두 번 쓰면 별개의 스트리밍 잡(Job)이 두 개 돌아가게 되어 자원을 낭비합니다. 
가장 효율적인 방법은 **동일한 가공 데이터(`output_df`)를 재사용**하는 것입니다.

---
## 다중 출력 시 반드시 지켜야 할 규칙 (Checkpoint)

매우 중요한 포인트입니다! 각 출력(Sink)은 **반드시 서로 다른 체크포인트 경로**를 가져야 합니다.

- **JSON용:** `chk/json`
- **Kafka용:** `chk/kafka`

**이유:** 체크포인트는 "내가 어디까지 읽었나"를 기록하는 수첩입니다. 
두 공장이 같은 수첩을 쓰면 서로 페이지를 넘기려다 싸움이 나고 데이터가 꼬이게 됩니다.


---
## 실전 코드 분석 (코드 조각)

### ① 데이터 가공 (Common Logic)

먼저 데이터를 읽어서 공통적으로 가공합니다. 
이 결과물(`concat_df`, `output_df`)이 두 군데로 나뉘어 나갑니다.

### ② 출력 1: 파일로 저장 (JSON Sink)

가공된 데이터를 로컬 폴더에 `json` 파일 형태로 차곡차곡 쌓습니다. 

```python
file_writer = concat_df \
    .writeStream \
    .format("json") \
    .option("path", "transformed") \
    .option("checkpointLocation", "chk/json") \
    .start()
```

### ③ 출력 2: 카프카로 전송 (Kafka Sink)

동일한 데이터를 카프카가 읽을 수 있게 `to_json`으로 다시 포장하여 전송합니다.

```python
kafka_writer = output_df \
    .writeStream \
    .format("kafka") \
    .option("topic", "transformed") \
    .option("checkpointLocation", "chk/kafka") \
    .start()
```

---
