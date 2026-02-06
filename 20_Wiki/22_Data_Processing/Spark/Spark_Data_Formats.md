---
aliases:
  - 스파크 파일 포맷
  - Parquet vs Avro
  - ORC
  - 데이터 저장 포맷
tags:
  - Spark
related:
  - "[[Spark_DataFrame_Basics]]"
  - "[[Spark_Streaming_Kafka_Integration]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약

**스파크 데이터 포맷**이란 데이터를 디스크에 저장하거나 읽어올 때 사용하는 **"파일의 구조와 규칙"** 으로, 용도(분석용 vs 전송용)에 따라 
**Parquet(분석 대장)**, **Avro(전송 대장)**, **ORC**, **JSON** 등을 골라 써야 합니다.

---
## Why: 왜 포맷을 골라 써야 할까?

* **문제점:** "그냥 익숙한 JSON이나 CSV 쓰면 안 되나요?" 🙅‍♂️
    * **JSON/CSV**는 사람이 읽기는 좋지만, 용량이 뻥튀기되고(압축 효율 ↓) 읽을 때마다 텍스트를 파싱하느라 속도가 엄청 느립니다. 
    * 빅데이터 환경에서는 이게 다 **돈(Storage 비용)과 시간(처리 속도)** 입니다.
* **해결책:**
    * 분석할 땐 **컬럼 기반(Parquet)** 으로 저장해서 필요한 열만 쏙 뽑아 읽고,
    * 스트리밍할 땐 **로우 기반(Avro)** 으로 저장해서 스키마 변경에 대응해야 합니다.

---
## Practical Context: 실무에서는 언제 뭘 쓸까?

상황별로 딱 정해드릴게요.

1.  **데이터 웨어하우스 / 분석 (OLAP):** **무조건 Parquet**.
    * *상황:* "지난달 매출액 합계 구해줘." (특정 컬럼만 필요함)
    * Spark의 기본 포맷이기도 하고, 압축률이 좋아서 조회 속도가 가장 빠릅니다.

2.  **카프카 연동 / 실시간 파이프라인:** **Avro**.
    * *상황:* "사용자 로그 포맷이 내일부터 바뀐대." (스키마 변경)
    * **Schema Evolution(스키마 진화)** 기능 덕분에 필드가 추가되어도 에러가 안 납니다.

3.  **외부 API 데이터 수집:** **JSON**.
    * *상황:* "웹 서버에서 로그 받아왔어."
    * 일단 JSON으로 받고, 받자마자 **Parquet로 변환해서 저장**하는 게 국룰입니다.

4.  **Hive 중심 환경:** **ORC**.
    * Hadoop/Hive를 메인으로 쓴다면 Parquet보다 성능이 조금 더 좋을 수 있습니다.
---
## Code Core Points

모든 포맷은 `read.format()`과 `write.format()`이라는 통일된 문법을 씁니다.

* **읽기:** `{python}spark.read.format("포맷").load("경로")` (또는 `spark.read.parquet("경로")`)
* **쓰기:** `{python}df.write.format("포맷").save("경로")` (또는 `df.write.parquet("경로")`)

---
## Detailed Analysis
각 포맷의 특징을 뜯어봅시다.

### ① Parquet (파케이) - 분석의 제왕

```python
# [핵심] 컬럼 기반(Columnar)이라서 필요한 '가격' 컬럼만 읽어옴 (I/O 절약)
df = spark.read.parquet("/data/sales_log")
```

- **장점:** 컬럼 단위 저장이라 압축이 엄청 잘 됨. `df.select("col1")` 할 때 속도 최강
- **단점:** 쓰는 속도(Write)는 좀 느림(압축하느라). 그리고 바이너리라 `cat` 명령어로 못 읽음.

### ② Avro (에이브로) - 유연함의 대명사

```python
# [핵심] 로우 기반(Row-oriented)이라 한 줄씩 쓰고 읽는 건 빠름
df = spark.read.format("avro").load("/data/user_logs")
```

- **장점:** **스키마 진화(Schema Evolution)**. 데이터 구조가 바뀌어도 옛날 데이터랑 같이 읽을 수 있음.
- **단점:** 특정 컬럼만 뽑아내는 분석 쿼리는 Parquet보다 느림.

### ③ JSON - 호환성 셔틀

```python
# [핵심] 텍스트 기반이라 용량 크고 느림. 디버깅용으로만 추천.
df = spark.read.json("/data/raw_logs.json")
```

- **장점:** 눈으로 바로 보임(`cat`). 어디서든 다 지원함.
- **단점:** 스키마가 없어서 데이터 꼬이면 에러 남. 압축 안 돼서 용량 돼지.

---
## 초보자들 흔한실수

- **"JSON이 제일 편하니까 계속 JSON으로 저장하면 안 돼요?"**    
    - 큰일 납니다. 데이터가 GB 단위만 넘어가도 읽는 속도가 10배 이상 차이 납니다. 
    - "저장은 Parquet, 확인은 JSON" 원칙을 지키세요.

- **"Parquet 파일 열었는데 글자가 깨져요!"**
    - 정상입니다. 압축된 바이너리 파일이라 텍스트 에디터로 못 봅니다. 
    - 확인하려면 스파크에서 `df.show()`로 보거나 전용 뷰어 도구를 써야 합니다.

- **"ORC는 언제 써요?"**
    - 회사에서 "우리는 Hive 쓴다"라고 할 때만 쓰세요. 
    - 그 외엔 Spark 생태계에서 Parquet가 더 표준에 가깝습니다.