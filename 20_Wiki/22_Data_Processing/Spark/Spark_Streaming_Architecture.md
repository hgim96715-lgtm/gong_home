---
aliases:
  - Spark Streaming Sources
  - Triggers
  - Micro-batch
tags:
  - Spark
  - Streaming
related:
  - "[[Spark_Streaming_Intro]]"
  - "[[Streaming_Source_Comparison]]"
  - "[[00_Apache_Spark_HomePage]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step24.ipynb
---
## Input Sources (데이터는 어디서 오는가?) 

스파크 스트리밍이 데이터를 읽어올 수 있는 대표적인 소스(Source)들입니다

| 소스 (Source) | 용도 및 특징                                                                                                                          |
| :---------- | :------------------------------------------------------------------------------------------------------------------------------- |
| **Socket**  | **학습 및 테스트용.**  <br> `nc`(Netcat) 등을 이용해 텍스트 데이터를 전송합니다. 실무 사용 금지.                                                               |
| **Rate**    | **벤치마킹용.** <br> 초당 정해진 개수의 가짜 데이터를 생성해 스파크 클러스터 성능을 테스트할 때 씁니다.                                                                  |
| **File**    | **파일 감지.** <br> 디렉토리에 새로운 파일(CSV, JSON, ORC, Parquet 등)이 들어오면 감지해서 읽습니다.<br> ※ 주의: 파일은 이동(Move) 등으로 원자적(Atomically)으로 생성되어야 합니다. |
| **Kafka**   | **실무 표준.** <br> 가장 많이 사용되는 메시지 브로커 소스입니다.                                                                                        |

---
##  Processing Model (어떻게 처리하는가?) 

스파크 스트리밍은 데이터를 물처럼 계속 흘려보내는 것이 아니라, **아주 작은 덩어리(Micro-Batch)로 잘라서 처리**합니다. 

###  처리 흐름 (Read -> Process -> Write)

1.  **Read:** 소스에서 데이터를 읽어옵니다.
2.  **Process:** 스파크 SQL 엔진을 사용해 데이터를 가공합니다 (예: 단어 쪼개기 `split`, 카운트 `count`).
3. **Write:** 결과를 저장소(Sink)나 화면에 출력합니다. 

이 모든 과정은 백그라운드 스레드(Background Streaming Thread)에서 끊임없이 반복됩니다

---
##  Trigger Settings (언제 실행하는가?) 

"데이터를 얼마나 자주 처리할 것인가?"를 결정하는 중요한 설정입니다.

### ① Unspecified (기본값)

* **동작:** 이전 배치가 끝나면 **즉시** 다음 배치를 시작합니다. 
* **지연 시간:** 마이크로 배치 모드로 동작하며 약 100ms 정도의 지연이 발생합니다. 

### ② Fixed Interval (고정 주기)

* **동작:** 사용자가 지정한 시간 간격(예: "10 seconds")마다 배치를 실행합니다. 
    * **처리가 빨리 끝나면?** -> 다음 주기까지 **대기**합니다. 
    * **처리가 늦어지면?** -> 끝나자마자 **즉시** 시작합니다. 

### ③ Available-now (구 One-time)

* **동작:** 현재 소스에 남아있는 모든 데이터를 처리하고 **스스로 종료**합니다. 
* **용도:** 주기적으로 실행되는 배치 작업(Airflow 등)에서 스트리밍 코드를 재활용할 때 유용합니다.

### ④ Continuous (실험적 기능)

* **동작:** 마이크로 배치가 아니라 진짜 스트리밍처럼 계속 처리합니다.
* **특징:** **~1ms** 수준의 초저지연(Low Latency)을 제공하지만 제약 사항이 많습니다. 

---
### 코멘트

"처음엔 무조건 **기본값(Unspecified)** 으로 시작해. 그러다 데이터가 너무 쌓여서 배치가 밀리면 주기를 조절하거나 클러스터를 늘리는 거야.
특히 **File Source** 쓸 때는, 파일을 다 쓰고 나서 폴더로 옮기는(Move) 방식을 써야 스파크가 깨진 파일을 읽지 않아!"

---
##  파일 소스 & 트리거 예제 (File Source Example) 

위에서 배운 **File Source**와 **Trigger** 설정을 실제 코드로 구현한 예제입니다.
`data/streaming_sample` 폴더에 JSON 파일이 들어오면 감지해서 처리합니다.

```python
from pyspark.sql import SparkSession
import os
# ---------------------------------------------------------------
# [Tip] 테스트용 데이터 생성 명령어 (터미널)
# fakedata --format=ndjson --limit 10000 city domain event=event.action > streaming_sample/sample.json
# ---------------------------------------------------------------

# 0. [필수] 관찰할 폴더 미리 만들기 (안 만들면 에러 남!)
# 스파크가 시작될 때 이 폴더가 없으면 "Path does not exist" 에러가 발생합니다.
if not os.path.exists("data/streaming_sample"):
	os.makedirs("data/streaming_sample")
	print("✅ data/streaming_sample 폴더 생성 완료!")


# 1. 세션 생성 및 설정 (Config)
spark = SparkSession \
    .builder \
    .appName("StructuredStreamingSum") \
    .config("spark.streaming.stopGracefullyOnShutdown", "true") \
    .config("spark.sql.streaming.schemaInference", "true") \
    .config("maxFilesPerTrigger", 1) \
    .getOrCreate()

# 2. 읽기 (Read): File Source
# "streaming_sample" 폴더를 감시하다가 JSON 파일이 생기면 읽습니다. 
df = spark \
    .readStream \
    .format("json") \
    .option("path", "data/streaming_sample") \
    .load()

# 3. 가공 (Process)
# 필요한 컬럼만 선택
shorten_df = df.select("city", "event")

# 4. 쓰기 (Write): File Sink & Trigger
query = shorten_df \
            .writeStream \
            .format("json") \ # 화면에 보고 싶으면 "console"로 변경
            .option("path", "data/streaming_output") \
            .option("checkpointLocation", "checkpoint") \
            .outputMode("append") \
            .trigger(processingTime='5 seconds') \
            .start()

query.awaitTermination()
```

---
### 코드 뜯어보기 (핵심 옵션)

1. **`{python}spark.streaming.stopGracefullyOnShutdown`, `true`**

	- **의미:** "스파크야, 종료 신호(SIGTERM)를 받으면 **하던 일(배치)은 마저 끝내고** 퇴근해라."
	- **왜 쓰나요?** 이 옵션이 없으면 종료 버튼을 누르는 순간 작업 중이던 데이터가 증발할 수 있습니다. (강제 종료 방지)
	- **비유:** 컴퓨터 끄기 전에 "저장하시겠습니까?" 묻고 안전하게 끄는 것과 같습니다.

2. **`{python}spark.sql.streaming.schemaInference`, `true`**

    - 원래 스트리밍은 스키마(컬럼 구조)를 미리 정해줘야 하지만, 이 옵션을 켜면 파일 내용을 보고 **스파크가 알아서 추측**합니다. (편리함!)

3. **`{python}maxFilesPerTrigger`, `1`**

    - **"한 번에 파일 1개씩만 처리해!"** 라고 제한을 겁니다.
    - 갑자기 파일 100개가 쏟아져도 1개씩 차근차근 처리하므로 **부하 조절(Throttling)** 에 유용합니다.
        
4. **`{python}trigger(processingTime='5 seconds')`**

    - **Fixed Interval Trigger**입니다. 5초마다 새로운 파일이 있는지 확인하고 배치를 실행합니다.

5. **`{python}option("checkpointLocation", ...)`**

    - **"어디까지 읽었는지 책갈피 저장."**
    - 스파크가 중간에 죽었다 살아나도, 이 체크포인트를 보고 **안 읽은 파일부터 다시 시작**할 수 있습니다. (파일 소스 필수 옵션!)

---
## 파일 소스(File Source) 사용 시 흔한 실수 

File Source는 사용하기 쉽지만, 처음 사용할 때 가장 많이 겪는 **두 가지 문제**가 있습니다. 
데이터가 안 들어오거나, 무시되는 상황을 피하려면 아래 내용을 꼭 숙지하세요.

### ① 데이터 추가 시 주의사항 (Append X, New File O)

- **문제 상황:** `sample.json` 파일에 데이터를 계속 추가했는데 스파크가 읽지 않음.
- **원인:** 스파크는 파일의 **"이름"이 새로워야** 새로운 데이터로 인식합니다. 이미 처리한 파일(`sample.json`)에 내용만 추가하면 "이미 본 파일"이라며 무시합니다.
- **해결책:** 데이터를 추가할 때는 항상 **새로운 파일명**(`sample_1.json`, `sample_2.json`...)으로 생성해야 합니다.

### ② 체크포인트(Checkpoint)의 기억력

- **문제 상황:** 코드를 멈췄다가 다시 실행했는데, 기존에 만들어둔 파일들을 읽지 않음.
- **원인:** `checkpoint` 폴더에 **"나 여기까지 읽었음"** 이라는 기록이 남아있기 때문입니다.
- **해결책:** 테스트할 때 처음부터 다시 읽게 하려면 **체크포인트 폴더와 결과 폴더를 삭제(Reset)** 해야 합니다.

---
## 실습용 도구 모음 (Toolbox) 

실습할 때 꼭 필요한 **"기억 초기화 도구"** 와 **"데이터 생성 도구"** 입니다.

### 초기화 도구 (Reset Script)

스파크를 멈추고 다시 처음부터 테스트하고 싶을 때 실행하세요.

```python
import shutil
import os

# 1. 체크포인트 삭제 (기억 지우기)
if os.path.exists("checkpoint"):
    shutil.rmtree("checkpoint")
    print("✅ 체크포인트 삭제 완료 (기억 리셋)")

# 2. 결과 폴더 삭제 (새 출발)
if os.path.exists("data/streaming_output"):
    shutil.rmtree("data/streaming_output")
    print("✅ 결과 폴더 삭제 완료 (새 출발)")
```

### streaming_sample폴더만 남겨두고 파일 .json없애고 싶다면 리눅스 명령어로 한 방에 (`rm`)

```bash
# data/streaming_sample 안에 있는 모든 것(*)을 강제로(-f) 재귀적으로(-r) 삭제
!rm -rf data/streaming_sample/*

print("🧹 청소 끝!")
```




### 데이터 생성기 (Data Generator)

실행할 때마다 **시간(Timestamp)** 을 파일명에 붙여서, 스파크가 항상 새로운 파일로 인식하게 만드는 코드입니다.

```python
import json
import random
import os
import time

# 1. 저장 경로
directory = "data/streaming_sample"
if not os.path.exists(directory):
    os.makedirs(directory)

# 2. 재료 준비
cities = ["Seoul", "New York", "London", "Paris", "Tokyo", "Busan"]
events = ["view", "click", "purchase", "login", "logout", "add_to_cart"]
domains = ["google.com", "naver.com", "yahoo.com", "facebook.com", "medium.com"]

# 3. ★ 핵심: 파일 이름에 시간을 붙임 (Unique Filename)
# 예: sample_1706934212.json
current_time = int(time.time())
file_path = f"{directory}/sample_{current_time}.json"

print(f"📂 '{directory}' 폴더에 새로운 파일을 생성 중입니다...")

# 4. 파일 쓰기
with open(file_path, "w", encoding="utf-8") as f:
    for _ in range(100):
        data = {
            "city": random.choice(cities),
            "domain": random.choice(domains),
            "event": random.choice(events)
        }
        f.write(json.dumps(data) + "\n")

print(f"✅ 생성 완료! 이번 파일: sample_{current_time}.json")
print("👉 스파크가 이 '새로운 파일'을 감지하고 가져갈 것입니다!")
```