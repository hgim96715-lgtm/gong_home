---
aliases:
  - Row
  - Column
  - SparkSession
  - SparkContext
  - RDD
  - spark.sparkContext
  - Row객체
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_DataFrame]]"
---

# Spark Core Objects — SparkSession / Row / Column / RDD

## 한 줄 요약

> **SparkSession = 출입문 / Row = 행 하나 / Column = 열 하나 / RDD = DataFrame 아래 층.**

---

---

# ① SparkSession — 모든 것의 시작점

```
Spark 를 사용하려면 반드시 SparkSession 을 먼저 만들어야 한다.
DataFrame 생성, SQL 실행, 파일 읽기 등 모든 작업의 진입점.
```

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("TrainConsumer")              # 앱 이름 (Spark UI 에 표시)
    .master("local[2]")                    # 로컬 2코어 (테스트용)
    .config("spark.sql.shuffle.partitions", "2")  # 설정값 추가
    .getOrCreate()                         # 이미 있으면 재사용, 없으면 생성
)
```

```
.master() 값:
  local        → 코어 1개 (싱글 스레드)
  local[2]     → 코어 2개
  local[*]     → 가용 코어 전부
  spark://host:7077  → Standalone 클러스터

getOrCreate():
  같은 JVM 안에서 이미 SparkSession 이 있으면 새로 만들지 않고 재사용
  노트북에서 셀을 여러 번 실행해도 중복 생성 안 됨
```

## 자주 쓰는 설정

```python
spark = (
    SparkSession.builder
    .appName("MyApp")
    .config("spark.sql.shuffle.partitions", "2")   # 기본 200 → 소규모 프로젝트는 2~4 권장
    .config("spark.executor.memory", "1g")          # Executor 메모리
    .getOrCreate()
)

spark.sparkContext.setLogLevel("WARN")   # INFO 로그 억제 (터미널 깔끔하게)
```

## 종료

```python
spark.stop()   # 프로그램 끝날 때 반드시 호출
               # finally 블록에 넣는 것이 안전
```

---

---

# ② Row — 행(데이터 한 줄)을 표현하는 객체

```
DataFrame 의 한 행(Row) 을 나타내는 객체.
딕셔너리처럼 키로 접근하거나 인덱스로도 접근 가능.
```

## 기본 사용법

```python
from pyspark.sql import Row

# Row 생성 — 키워드 인자로 컬럼명 지정
row = Row(trn_no="KTX001", dptre="서울", arvl="부산", delay=10)

# 접근 방법
print(row.trn_no)    # "KTX001"  ← 속성처럼 접근 (권장)
print(row["trn_no"]) # "KTX001"  ← 딕셔너리처럼 접근
print(row[0])        # "KTX001"  ← 인덱스로 접근 (비추천 — 순서에 의존)
```

## Row → DataFrame 변환

```python
rows = [
    Row(trn_no="KTX001", dptre="서울", arvl="부산", delay=10),
    Row(trn_no="KTX002", dptre="서울", arvl="대전", delay=3),
]

df = spark.createDataFrame(rows)
# Row 의 키워드 인자 이름이 자동으로 컬럼명이 됨
df.show()
```

```
+-------+-----+----+-----+
| trn_no|dptre|arvl|delay|
+-------+-----+----+-----+
|KTX001 |서울 |부산 |   10|
|KTX002 |서울 |대전 |    3|
+-------+-----+----+-----+
```

## 비정형 데이터 파싱 패턴 (parse_line)

로그 파일, 파이프 구분 텍스트 같은 비정형 데이터를 Row 로 변환하는 패턴.

```python
from pyspark.sql import Row

raw_lines = [
    "KTX001|서울|부산|10",
    "KTX002|서울|대전|3",
]

def parse_line(line: str) -> Row:
    fields = line.split("|")
    return Row(
        trn_no=fields[0],
        dptre=fields[1],
        arvl=fields[2],
        delay=int(fields[3]),   # 타입 변환도 여기서
    )

rdd  = spark.sparkContext.parallelize(raw_lines)
rows = rdd.map(parse_line)      # 모든 줄에 parse_line 적용
df   = spark.createDataFrame(rows)
```

```
Row 를 쓰는 이유:
  fields[0] 으로 접근하면 나중에 봤을 때 뭔지 모름
  Row(trn_no=fields[0]) 으로 이름을 붙이면 의미가 명확해짐
  createDataFrame 할 때 키가 자동으로 컬럼명이 됨
```

## DataFrame → Row 꺼내기

```python
# collect() — 전체를 드라이버로 가져옴 (소규모 데이터에서만 사용)
rows = df.collect()           # List[Row] 반환
first = rows[0]
print(first.trn_no)           # "KTX001"

# first() — 첫 번째 행만
row = df.first()
print(row["trn_no"])

# take(n) — 상위 n개
rows = df.take(3)             # List[Row] 반환
```

```
⚠️ collect() 주의사항:
  전체 데이터를 드라이버 메모리로 가져옴
  데이터가 크면 드라이버 OOM (메모리 부족) 발생
  수백만 건 이상이면 절대 사용 금지
  테스트 / 소규모 결과 확인 용도로만 사용
```

---

---

# ③ Column — 열을 표현하는 객체

```
DataFrame 의 한 열(Column) 을 나타내는 객체.
select, filter, withColumn 등 대부분의 변환에서 Column 객체를 사용.
```

## Column 참조 방법 3가지

```python
from pyspark.sql.functions import col

df.select("trn_no")          # 방법 1: 문자열 (간단)
df.select(col("trn_no"))     # 방법 2: col() 함수 (표현식 사용 가능)
df.select(df["trn_no"])      # 방법 3: DataFrame["컬럼명"] (명확하지만 장황)
```

```
언제 col() 을 쓰나:
  단순 조회       → "trn_no" 문자열로 충분
  조건/계산식     → col() 필요

  col("delay") > 5         # 조건
  col("delay") * 60        # 계산
  col("trn_no").alias("열차번호")  # 별칭
```

## alias() — 컬럼 이름 바꾸기

```python
df.select(
    col("trn_no").alias("열차번호"),
    col("delay").alias("지연분"),
).show()
```

## cast() — 타입 변환

```python
from pyspark.sql.functions import col

# 문자열 → 정수로 변환
df = df.withColumn("delay", col("delay").cast("integer"))

# cast 에 넣을 수 있는 타입 문자열
# "string", "integer", "long", "double", "boolean", "timestamp", "date"
```

---

---

# ④ RDD — DataFrame 아래 층

```
RDD (Resilient Distributed Dataset)
  스파크의 가장 기본적인 데이터 구조.
  DataFrame 이 내부적으로 RDD 위에서 동작함.

실무에서는 거의 DataFrame / Dataset 사용
RDD 를 직접 쓰는 경우:
  비정형 텍스트 파싱 (위의 parse_line 패턴)
  DataFrame API 로 표현하기 어려운 커스텀 로직
```

## sparkContext vs SparkSession

```python
# SparkSession — DataFrame API 진입점
spark = SparkSession.builder.appName("App").getOrCreate()

# SparkContext — RDD API 진입점 (SparkSession 안에 포함됨)
sc = spark.sparkContext
```

## 기본 사용법

```python
sc = spark.sparkContext

# 리스트 → RDD
rdd = sc.parallelize([1, 2, 3, 4, 5])

# 변환 (Transformation) — 새로운 RDD 반환, 즉시 실행 안 됨 (Lazy)
rdd2 = rdd.map(lambda x: x * 2)        # 각 요소에 함수 적용
rdd3 = rdd.filter(lambda x: x > 2)     # 조건으로 필터링
rdd4 = rdd.flatMap(lambda x: [x, x*2]) # 1개 → 여러 개로 펼치기

# 실행 (Action) — 실제 계산 시작
rdd2.collect()   # 전체 결과를 드라이버로 수집
rdd2.count()     # 요소 개수
rdd2.first()     # 첫 번째 요소
rdd2.take(3)     # 상위 3개
```

## RDD ↔ DataFrame 변환

```python
# RDD → DataFrame
df = spark.createDataFrame(rdd, schema=["value"])

# DataFrame → RDD
rdd = df.rdd
```

---

---

# 치트시트

```python
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import col

# SparkSession 생성
spark = SparkSession.builder.appName("App").master("local[2]").getOrCreate()
spark.sparkContext.setLogLevel("WARN")

# Row 생성
row = Row(name="KTX001", delay=10)
print(row.name)       # 속성 접근
print(row["name"])    # 딕셔너리 접근

# Row 리스트 → DataFrame
df = spark.createDataFrame([Row(name="A", val=1), Row(name="B", val=2)])

# Column 사용
df.select(col("name").alias("열차명"), col("val").cast("double"))

# SparkContext (RDD 용)
sc = spark.sparkContext
rdd = sc.parallelize([1, 2, 3])

# 종료
spark.stop()
```