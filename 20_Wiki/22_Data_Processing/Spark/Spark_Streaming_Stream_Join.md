---
aliases:
  - Stream-Stream Join
  - 실시간 조인
  - Double Stream Join
tags:
  - Spark
  - Streaming
related:
  - "[[Spark_Streaming_Watermark]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/kafka_streaming_join.py
---
## 개요: 흐르는 강물 두 줄기 합치기 

**Stream-Stream Join**은 두 개의 실시간 데이터 스트림을 특정 키(Key)와 시간(Time)을 기준으로 합치는 작업입니다.

* **상황:**
    * **Stream A (광고 노출):** 사용자에게 광고가 보임 (Impression).
    * **Stream B (광고 클릭):** 사용자가 광고를 클릭함 (Click).

* **목표:** "광고를 보고 나서 10분 안에 클릭한 경우"를 찾아서 과금 처리하기.

---
## 동작 원리: 기다림의 미학 (State Store) 

두 데이터가 정확히 동시에 도착할 확률은 0에 가깝습니다. 클릭은 항상 노출보다 늦게 오죠.

1.  **버퍼링 (Buffering):** 스파크는 먼저 도착한 데이터를 **State Store(메모리 저장소)** 에 임시로 보관합니다.
2.  **기다림:** 짝꿍 데이터가 들어올 때까지 기다립니다.
3.  **매칭:** 짝이 들어오면 조인 결과를 내보냅니다.

> **⚠️ 문제점:** 짝이 영원히 안 온다면? 메모리가 폭발(OOM)합니다.
> **✅ 해결책:** 그래서 **Watermark(워터마크)** 가 필수입니다.("30분 지나도 짝 안 오면 버려!")

---
## 필수 조건: 워터마크와 시간 제약 

Stream-Stream Join이 성공하려면 **두 가지 제약**을 반드시 코드에 넣어야 합니다.

### ① 양쪽 모두 워터마크 설정 (Watermark)

두 스트림 모두 "늦게 온 데이터는 언제 버릴지" 기준이 있어야 합니다.

### ② 조인 조건에 시간 범위 추가 (Time Constraint)

"무작정 기다리는 게 아니라, **1시간**까지만 기다리겠다"는 범위가 필요합니다.

```python
# 예: Click 시간은 Impression 시간 이후여야 하고, 최대 1시간까지만 인정한다.
"ImpressionTime <= ClickTime AND ClickTime <= ImpressionTime + 1 hour"
```

---
## 조인 종류별 특징

### Inner Join (교집합)

- **동작:** 광고도 보고, 클릭도 한 경우만 결과로 나옴.
- **출력 시점:** 짝꿍(Match)이 발견되는 **즉시** 출력됩니다.
- **워터마크:** 선택 사항이지만, State 관리를 위해 강력 권장.

### Outer Join (Left/Right)

- **동작:** "광고는 봤는데 클릭을 안 한 사람"도 결과로 남겨야 할 때 (로그 분석).
- **출력 시점:** **지연 출력됨.**
	- 스파크는 "혹시 늦게라도 클릭이 올까 봐" **워터마크 시간만큼 기다렸다가**, 끝까지 안 오면 그때 비로소 `null`을 채워서 내보냅니다.
- **특징:** **워터마크가 필수**입니다. (없으면 영원히 기다림)

----
## [실습] 핵심 로직 패턴

> 🔗 **전체 실습 코드:** [stream_join.py](file:///Users/gong/gong_study_de/apache-spark/notebooks/kafka_streaming_join.py) (클릭해서 파일 확인)

반복되는 읽기/파싱 로직은 함수로 분리하고, 조인은 깔끔하게 수행하는 패턴입니다.

### ① 파싱 및 워터마크 설정 함수 (재사용)

```python
def parse_stream(df, schema, watermark_duration):
    return (
        df.select(from_json(col("value").cast("string"), schema).alias("value"))
          .select("value.*")
          .withColumn("create_date", to_timestamp("create_date", "yyyy-MM-dd HH:mm:ss"))
          .withWatermark("create_date", watermark_duration) # 👈 핵심: 워터마크 필수!
    )
```

### ② 데이터 준비 및 조인 수행

```python
# 1. 데이터 준비 (함수 재사용)
impressions = parse_stream(read_kafka("impression"), imp_schema, "30 minutes")
clicks      = parse_stream(read_kafka("click"),      click_schema, "10 minutes")

# 2. 조인 수행 (상황별 문법 주의!)

# Case A: 단순히 ID만 같으면 될 때 (추천 )
# 장점: 중복 컬럼 자동 제거됨
join_df = impressions.join(clicks, on=['uuid'], how='inner')

# Case B: 시간 범위 제약이 필요할 때 (복잡)
# 주의: on 사용 불가 -> expr 사용 필수 -> 중복 컬럼 수동 제거 필요
# join_df = impressions.join(clicks, expr("uuid = uuid AND click_time >= imp_time ..."), "inner")
```

## 심화: 조인 문법 주의사항 

> [!tip] 💡 시간 조건이 들어갈 때 `on`을 못 쓰는 이유
>  **`on=['uuid']`** 문법은 오직 **"값이 똑같은가? (`{text}==`)"** 만 검사할 수 있습니다.
> 스트림 조인 필수 조건인 **"시간 범위 (`<=`, `BETWEEN`)"** 를 넣으려면 `on`을 포기하고 **`expr`**을 써야 합니다
> 👉 **상세 해결법 및 예제:** [[Spark_DataFrame_Joins#예외 시간 조건(Interval)을 넣을 때는 `on` 못 씀!|시간 조건 사용 시 on을 못 쓰는 이유 보기]]
