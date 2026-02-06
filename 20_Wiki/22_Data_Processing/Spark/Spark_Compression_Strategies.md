---
aliases:
  - 스파크 압축 알고리즘
  - Snappy vs Gzip
  - Splittable
  - 파일 압축
tags:
  - Spark
related:
  - "[[Spark_Data_Formats]]"
  - "[[Spark_Data_IO]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약

데이터 압축(Compression)은 단순히 파일 크기를 줄여 **저장 비용을 아끼고 네트워크 전송 속도를 높이는** 기술입니다. 
과거에는 속도(Snappy)와 용량(Gzip) 사이에서 양자택일해야 했지만, 최근 **ZSTD(Zstandard)** 의 등장으로 **"속도와 압축률의 밸런스"** 를 모두 잡는 추세입니다

---
## Why: 압축은 공짜 점심이 아니다 (Trade-off)

압축을 하면 무조건 좋을 것 같지만, **잃는 것(Cost)** 도 분명히 있습니다.

**장점 (Pros):**
- 저장 공간 절약 (돈 아낌)
- 네트워크 대역폭 절약 (데이터가 작으니 빨리 보냄)
- I/O 병목 해소 (디스크에서 읽는 시간 단축)

**단점 (Cons):**
- **CPU 부하 (Overhead):** 압축을 풀 때 CPU가 일을 해야 하므로 처리 속도가 느려질 수 있음
- **Splittability (분할 가능성):** 특정 압축 방식(Gzip 등)은 파일을 쪼개서 병렬 처리를 못 함.

---
##  Practical Context: 3파전 (Snappy vs Gzip vs ZSTD)

실무에서는 **"속도(Speed)"**, **"용량(Size)"**, **"밸런스(Balance)"** 중 무엇이 중요한지에 따라 결정합니다.

### 🥊 비교 분석

| 특징                 | **Snappy** (Google)                          | **Gzip**                               | **ZSTD** (Facebook)                          |
| :----------------- | :------------------------------------------- | :------------------------------------- | :------------------------------------------- |
| **핵심 키워드**         | **Ultra Fast**                               | **High Ratio**                         | **Sweet Spot (Balance)**                     |
| **속도**             | 압축/해제 매우 빠름           | 느림         | **빠름** (Snappy급)      |
| **압축률(용량)**        | 낮음 (용량 큼)             | **높음** (용량 작음)  | **높음** (Gzip급)        |
| **분할(Splittable)** | **Yes** (Parquet 짝꿍)  | **No** (기본적으로 불가)          | **Yes**                                      |
| **추천 용도**          | 초저지연 실시간 처리           | 콜드 데이터(로그) 보관   | **데이터 레이크 표준 (분석용)**  |


>**Choice:**
>**기본 전략:** 고민할 것 없이 **Parquet + ZSTD** 조합을 쓰세요. 속도는 Snappy만큼 빠른데 용량은 Gzip만큼 줄여줍니다
>**예외 1:** 0.001초가 급한 초고속 스트리밍이라면 여전히 **Snappy**가 답입니다.
>**예외 2:** CPU가 터질 것 같고 스토리지는 널널하다면 **Gzip**이나 **Snappy**로 타협하세요.


---
## Key Concept: "쪼갈라지는가?" (Splittable) 

빅데이터에서 가장 중요한 개념입니다.

* ***Splittable (분할 가능):** 파일 하나를 여러 블록(Block)으로 나눠서, 여러 노드가 **동시에 병렬 처리**할 수 있습니다. (Snappy, ZSTD, LZ4) .
* **Non-Splittable (분할 불가):** 파일이 아무리 커도(100GB), **단 하나의 노드**가 처음부터 끝까지 혼자 처리해야 합니다. (Gzip) .
    * **결과:** 100대의 서버가 있어도 1대만 일하고 나머지는 놉니다. 성능 나락 확정.

---
## Code Core Points

`{python}.option("compression", "...")` 만 기억하면 됩니다.  스파크 3.x부터는 ZSTD를 기본 지원합니다.

### ① 개별 파일 저장 시

```python
# 1. ZSTD로 저장 (추천: 밸런스형)
df.write.option("compression", "zstd").parquet("/data/output_zstd")

# 2. Snappy로 저장 (속도 중시)
df.write.option("compression", "snappy").parquet("/data/output_snappy")

# 3. Gzip으로 저장 (용량 중시)
df.write.option("compression", "gzip").parquet("/data/output_gzip")
```

### ② 전역 설정 (Session Config) 꿀팁

매번 옵션 넣기 귀찮다면, 스파크 세션을 만들 때 기본값으로 박아두세요.

```python
spark = SparkSession.builder \
    .config("spark.sql.parquet.compression.codec", "zstd") \
    .config("spark.sql.orc.compression.codec", "zstd") \
    .getOrCreate()
```

## Detailed Analysis (튜닝 레벨)

ZSTD의 강력한 무기는 **"압축 레벨 조절(Tunable)"** 이 가능하다는 점입니다.

- **Level 1~3 (기본):** 속도 중시. 쓰기 작업이 많을 때 추천.
- **Level 6~9:** 밸런스.
- **Level 19+:** 극강의 압축. CPU를 갈아 넣어 용량을 최소화합니다 (아카이빙용).

```python
# ZSTD 압축 레벨 설정 (기본값은 보통 3)
spark.conf.set("parquet.compression.codec.zstd.level", "6") 
```

---
## Detailed Analysis (요약표)

| **알고리즘**   | **속도 (Speed)** | **압축률 (Ratio)** | **분할 가능?** | **추천 용도 (Best Practice)**                |
| ---------- | -------------- | --------------- | ---------- | ---------------------------------------- |
| **Snappy** | **매우 빠름** ⚡️   | 낮음 (Low)        | **Yes**    | **Spark 기본값**, 지연 시간이 중요한 실시간 분석         |
| **LZ4**    | **극도로 빠름** 🚀  | 매우 낮음           | **Yes**    | **초저지연(Ultra-low latency)** 스트리밍, 임시 데이터 |
| **ZSTD**   | **빠름** (조절 가능) | **높음 (High)**   | **Yes**    | **데이터 레이크 표준**, 분석과 보관 두 마리 토끼 잡기        |
| **Gzip**   | 느림 (Slow)      | **높음 (High)**   | **No** ❌   | **장기 보관(Archival)**, 텍스트 로그(CSV/JSON) 백업 |


---
## 초보자들이 많이하는 착각 

- **"압축률이 높을수록(Gzip) 좋은 거 아닌가요?"**
    - 빅데이터 처리에서는 **아닙니다.** 
    - 압축을 푸느라 CPU를 다 써버리고, 병렬 처리도 안 되면 전체 작업 시간은 훨씬 늘어납니다.

- **"CSV도 Snappy로 압축하면 빨라지나요?"**
    - CSV나 JSON 같은 텍스트 파일은 Snappy와 궁합이 잘 안 맞거나 지원이 제한적일 수 있습니다. 
    - 보통 텍스트 파일은 Gzip을 많이 씁니다. 
    - Snappy는 **Parquet/Avro**와 짝꿍입니다.