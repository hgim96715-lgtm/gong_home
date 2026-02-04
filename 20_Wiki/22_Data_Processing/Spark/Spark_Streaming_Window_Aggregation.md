---
aliases:
  - Window Aggregation
  - Tumbling Window
  - Sliding Window
  - 윈도우 집계
tags:
  - Spark
  - Streaming
related:
  - "[[Spark_Streaming_Stateful_Stateless]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Data_Cleaning]]"
  - "[[Spark_Streaming_Watermark|늦게 도착한 데이터 처리(Watermark)]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/kafka_tumbling.py
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/kafka_sliding.py
---
##  개요: 무한한 스트림을 자르는 법 

스트리밍 데이터는 끝이 없습니다. 
그래서 "지난 1시간 동안의 매출"처럼 **시간(Time)을 기준으로 구역(Window)을 나누어** 집계해야 합니다. 이를 **Window Aggregation**이라고 합니다.

---
##  Window 함수 문법 (Syntax) 

스파크에서는 `groupBy` 안에 `window()` 함수를 사용하여 시간을 정의합니다.

```python
from pyspark.sql.functions import window

# 문법 구조
window(timeColumn, windowDuration, slideDuration=None, startTime=None)
```

|**파라미터**|**설명**|**필수**|**예시**|
|---|---|---|---|
|**`timeColumn`**|기준 시간 컬럼 (**반드시 Timestamp 타입**)|✅|`col("event_time")`|
|**`windowDuration`**|윈도우 크기 (길이)|✅|`"10 minutes"`, `"1 hour"`|
|**`slideDuration`**|윈도우 이동 간격 (생략 시 Tumbling)|❌|`"5 minutes"`|

> **⚠️ 주의:** `timeColumn`이 문자열(String)이라면 반드시 `to_timestamp()`로 변환 후 넣어야 합니다.
>  👉 `to_timestamp()` 사용법이 궁금하다면 **[[Spark_Data_Cleaning#🕒 날짜와 시간 타입 변환 (Date & Timestamp)|to_timestamp 변환 가이드]]** 를 참고

---
### 결과 데이터 구조 (Output Struct)

이 함수를 실행하면 생성되는 `window` 컬럼은 **`{start, end}` 형태의 구조체(Struct)** 로 표시됩니다.

- **형태:** `{python}StructType(start: Timestamp, end: Timestamp)`
    
- **접근 방법:** `col("window.start")` 또는 `col("window.end")`로 하위 필드에 접근할 수 있습니다.

```text
+------------------------------------------+------------+
|window                                    |total_amount|
+------------------------------------------+------------+
|{2026-02-04 00:10:00, 2026-02-04 00:15:00}|1800        |
+------------------------------------------+------------+
```

---
## Tumbling Time Window (텀블링 윈도우) 📦

시간이 **겹치지 않게(Non-overlapping)** 딱딱 끊어서 집계하는 방식입니다.

- **특징**:
    - 고정된 크기(Fixed-size)의 시간 간격으로 나눕니다.
    - 인접한 윈도우끼리 **절대 겹치지 않습니다**.
    - 윈도우가 닫히면(Expire) 그 안의 데이터는 독립적으로 처리되고 끝납니다.
- **비유**: 12시 00분~12시 05분, 12시 05분~12시 10분 처럼 딱 떨어지는 **기차 시간표**.
- 데이터의 **시간(Event Time)** 을 보고 어느 그릇(00~05분, 05~10분...)에 넣을지 결정합니다.
- **데이터 매핑**: 하나의 데이터는 오직 **하나의 윈도우**에만 들어갑니다.
- 데이터 1개 -> 윈도우 1개 (깔끔함, 정산용)


```python
# 10분 단위로 딱딱 끊어서 단어 카운트 (겹침 없음)
# window(시간, 크기)
windowedCounts = words \
    .groupBy(window(col("timestamp"), "10 minutes")) \
    .count()
```

---
## Sliding Time Window (슬라이딩 윈도우) 

시간이 **겹치면서(Overlapping)** 미끄러지듯이 집계하는 방식입니다.

- **특징**:
    - 고정된 크기(Fixed-size)이지만, 일정 간격(Slide Duration)마다 새 윈도우가 시작됩니다.        
    - 하나의 데이터가 **여러 윈도우에 중복**되어 포함될 수 있습니다.
    - **연속적인 패턴**이나 추세를 모니터링할 때 유용합니다.

- **예시**: "최근 1분간의 네트워크 트래픽을 30초마다 갱신해서 보여줘".
    - 윈도우 1: `12:00 ~ 12:01`
    - 윈도우 2: `12:00:30 ~ 12:01:30` (30초 뒤에 시작)

- **데이터 매핑**: 12:00:45에 들어온 데이터는 윈도우 1과 윈도우 2 **모두**에 포함됩니다.
- 데이터 1개 -> 윈도우 N개 (중복 발생, 트렌드 분석용)

```python
# "10분"짜리 윈도우를 "5분"마다 갱신 (5분 겹침)
# window(시간, 크기, 이동간격)
windowedCounts = words \
    .groupBy(window(col("timestamp"), "10 minutes", "5 minutes")) \
    .count()
```

### ② 데이터 매핑 예시 (Mapping Logic)

데이터 `00:07:00`이 들어왔을 때, 어떤 윈도우에 들어가는지 봅시다.
`{"create_date":"2026-02-04 00:07:00","amount": 1000}`

1. **윈도우 A (`00:00 ~ 00:10`):**    
    - 범위: 0분 이상 10분 미만
    - 판단: **7분은 포함됨!** ✅

2. **윈도우 B (`00:05 ~ 00:15`):**
    - 5분 뒤에 새로 열린 윈도우
    - 범위: 5분 이상 15분 미만
    - 판단: **7분은 포함됨!** ✅

3. **윈도우 C (`00:10 ~ 00:20`):**
    - 또 5분 뒤에 열린 윈도우
    - 판단: 7분은 포함 안 됨 ❌

### ③ 결과 확인 (Result)

**데이터는 1개(`00:07:00`)** 들어왔는데, **결과는 2줄**로 불어납니다.

```text
+------------------------------------------+------------+
|window                                    |total_amount|
+------------------------------------------+------------+
|{2026-02-04 00:00:00, 2026-02-04 00:10:00}| 300        |  <-- 여기도 300 포함
|{2026-02-04 00:05:00, 2026-02-04 00:15:00}| 300        |  <-- 저기도 300 포함 (중복!)
+------------------------------------------+------------+
```

---
## 6. 실전 실행 설정: Trigger와 Output Mode ⚙️

윈도우 집계 로직을 짰다면, 실제로 어떻게 실행할지 설정해야 합니다.

```python
query = (
    window_agg_df
    .writeStream
    .outputMode("update")                # 1. 출력 모드
    .format("console")                   # 2. 출력 대상 (Console, Kafka, Parquet 등)
    .option("truncate", "false")         # (Console용) 내용이 길어도 자르지 마라
    .trigger(processingTime="5 seconds") # 3. 실행 주기 (트리거)
    .start()
)
```

### `.trigger(processingTime="5 seconds")` 

- 이 설정이 없으면 스파크는 **"쉬지 않고(As fast as possible)"** 돕니다.
- **설정 시 (권장):** "5초 동안 데이터를 모았다가 한 번에 처리해!" (버스 배차 간격)
	- **장점:** 작은 파일이 무수히 생기는 문제(Small File Problem)를 막고, CPU 낭비를 줄입니다.
- **미설정 시:** 이전 작업이 끝나자마자 즉시 다음 작업을 시작합니다. (택시)

### `.outputMode("update")`

- 윈도우 집계에서는 **Update**나 **Complete** 모드를 써야 합니다. (Append는 Watermark 없이 사용 불가)
- **Update (실무용):** 집계 결과가 **변한 윈도우만** 출력합니다. (효율적)
- **Complete (디버깅용):** 모든 윈도우의 결과를 매번 **전부 다** 출력합니다. (데이터 많으면 터짐)

---
## 요약 비교 

|**구분**|**Tumbling Window**|**Sliding Window**|
|---|---|---|
|**핵심**|**겹치지 않음** (No Overlap)|**겹침** (Overlap)|
|**문법**|`window(time, size)`|`window(time, size, slide)`|
|**파라미터**|Window Size (1개)|Window Size + Slide Duration (2개)|
|**데이터 중복**|데이터는 오직 하나의 윈도우에만 속함|데이터가 여러 윈도우에 복제되어 들어갈 수 있음|
|**용도**|일별/시간별 정산|주식 이동평균선, 실시간 트래픽 감지|


---
## [실습] 텀블링 윈도우 결과 확인 (Console Output) 

실제로 데이터를 넣었을 때 스파크가 어떻게 시간을 묶어서 보여주는지 확인해 봅시다.

### ① 입력 데이터 (Input)

터미널(Producer)에 아래와 같이 5분 간격(00:10~00:15) 사이에 포함되는 데이터를 두 개 입력합니다.

```json
{"create_date":"2026-02-04 00:11:00","amount": 800}
{"create_date":"2026-02-04 00:13:00","amount": 1000}
```

### ② 윈도우 계산 로직 (Logic)

- **설정:** 5분 단위 Tumbling Window  :`window(col("create_date"),'5 minutes')`
    
- **계산:**
    - `00:11:00` 👉 **`00:10:00 ~ 00:15:00`** 구간에 포함
    - `00:13:00` 👉 **`00:10:00 ~ 00:15:00`** 구간에 포함
    - **결과:** 같은 구간이므로 합산 (`800 + 1000 = 1800`)

### ③ 스파크 출력 결과 (Output)

`window` 컬럼은 `{시작시간, 종료시간}` 형태의 구조체(Struct)로 표시됩니다.

```text
-------------------------------------------
Batch: 1
-------------------------------------------
+------------------------------------------+------------+
|window                                    |total_amount|
+------------------------------------------+------------+
|{2026-02-04 00:10:00, 2026-02-04 00:15:00}|1800        |
+------------------------------------------+------------+
```

