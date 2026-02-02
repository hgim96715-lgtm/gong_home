---
aliases:
  - Dynamic Partition Pruning
  - 파티션 가지치기
  - DPP
tags:
  - Spark
related:
  - "[[Spark_Partitioning_Concept]]"
  - "[[Spark_AQE_Deep_Dive]]"
  - "[[00_Apache_Spark_HomePage]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step18.ipynb
---
## 개념 한 줄 요약 

**"조인(Join)을 할 때, 상대방 테이블의 필터 조건을 보고 '내 파티션' 중 안 읽어도 되는 건 미리 버리는(Pruning) 기술."**

* **Static Partition Pruning (기존):** 쿼리에 `WHERE region = 'US'` 처럼 상수가 박혀 있을 때만 작동했습니다
* **Dynamic Partition Pruning (DPP):** `WHERE` 조건이 다른 테이블(Dimension Table)에 있을 때도, 그 결과를 보고 내 파티션을 동적으로 걸러냅니다

---
## 작동 원리 (Star Schema 최적화) 

주로 **Fact Table(대용량)** 과 **Dimension Table(소용량)** 을 조인할 때 발생합니다.

1.  **Dimension 필터링:** 작은 테이블(Dimension)에서 먼저 필터링을 수행합니다
2.  **Broadcasting:** 필터링된 키 값(지역 ID 등)을 모든 Executor에게 뿌립니다(Broadcast)
3.  **Pruning:** 큰 테이블(Fact)을 스캔할 때, 방송받은 키에 해당하지 않는 파티션은 **아예 파일을 열지도 않습니다(Skip)

---

##  도입 효과 (Benefits)

* **I/O 비용 급감:** 필요한 파티션만 콕 집어 읽기 때문에, 디스크 읽기(Scan) 양이 획기적으로 줄어듭니다.
* **쿼리 속도 향상:** 읽는 데이터가 줄어드니 당연히 전체 실행 시간도 빨라집니다.
* **복잡한 쿼리에도 적용:** 단순 필터뿐만 아니라, 서브쿼리나 조인(Join) 조건에서도 똑똑하게 작동합니다.

---
## [핵심] 물리적 파티셔닝(`partitionBy`)이란? 

DPP가 작동하려면 데이터가 디스크에 **"폴더별로 쪼개져서(Partitioned)"** 저장되어 있어야 합니다.

### 1. 파티셔닝을 안 했을 때 (일반 저장)

```python
df.write.parquet("/data/sales")
```

- **저장 구조:** `/data/sales/part-0001.parquet` (파일 하나에 region 1, 2, 3 데이터가 뒤섞여 있음)
- **문제:** "region 1만 가져와"라고 해도, 스파크는 파일 전체를 다 읽어서 뒤져봐야 합니다. (Full Scan)

### 2. 파티셔닝을 했을 때 (`partitionBy`)

```python
df.write.partitionBy("region_id").parquet("/data/sales_partitioned")
```

- **저장 구조:** **폴더**가 생깁니다
	- `/data/sales_partitioned/region_id=1/part-0001.parquet` (여기엔 1번 데이터만!)

- DPP의 원리:
	- 스파크: "어? 필터 조건이 `region_id=1`이네?"
	- **행동:** `region_id=2`와 `region_id=3` 폴더는 **아예 쳐다보지도 않고(Pruning)**, `region_id=1` 폴더만 엽니다.
	- **결과:** 읽어야 할 파일 양이 1/3로 줄어듭니다.

>**주의:** 코드에서 `df.write.partitionBy("col")`로 저장하지 않은 일반 테이블은 DPP가 절대 작동하지 않습니다!
>**파티셔닝된다는 뜻:** 데이터가 컬럼 값에 따라 **"물리적인 폴더(Directory)"** 로 나뉘어 저장된다는 뜻입니다.

---
## 실습: 코드로 확인하기 

DPP가 제대로 작동하는지 확인하는 예제입니다.

### ① 데이터 준비 (Partitioned Table 생성)

```python
from pyspark.sql import SparkSession, functions as F, types as t

# 1. 세션 생성
spark = (
    SparkSession
    .builder
    .appName("spark-DDP")
    .master("spark://spark-master:7077")
    .getOrCreate()
)

# 2. 버전 및 설정 확인 (Spark 3.x 이상 필수 / 기본값 True)
print(f"Spark Version: {spark.version}")
print(f"DPP Enabled: {spark.conf.get('spark.sql.optimizer.dynamicPartitionPruning.enabled')}")

# 3. Fact Table 데이터 준비 (CSV 로드)
table_schema = t.StructType([
    t.StructField("date", t.StringType(), True),
    t.StructField("name", t.StringType(), True),
    t.StructField("region", t.IntegerType(), True)])

csv_file_path = "file:///workspace/data/ecommerce_order.csv"
df = spark.read.schema(table_schema).csv(csv_file_path)

df.show()
df.printSchema()

# 4. 파티션 나누어 저장 (중요! 파티셔닝이 되어 있어야 Pruning이 됨)
(
    df
    .write
    .partitionBy("region")
    .mode("overwrite")
    .parquet("/workspace/data/output/partition_pruning")
)
```


### ② 비교: DPP가 아닌 경우 (Static Partition Pruning)

먼저, 저장된 데이터를 다시 읽어옵니다.

```python
# 데이터 다시 읽기 (Parquet)
read_df = spark.read.parquet("/workspace/data/output/partition_pruning")

# [비교] Static Partition Pruning (기존 방식)
# 쿼리에 WHERE 'region=2' 처럼 상수가 박혀있을 때만 작동합니다.
print("=== Static Partition Pruning ===")
read_df.where("region=2").explain("formatted")
```

**설명:** 이건 DPP가 아닙니다. 쿼리 짤 때 이미 `2`라고 못 박았기 때문에 스파크가 컴파일 타임에 압니다.

### ② DPP 적용: 조인 시 파티션 스킵 확인

런타임에 결정되는 조건(`San Francisco`에 해당하는 ID)으로 파티션을 거르는 진짜 DPP 예제입니다.

```python
# 5. Dimension Table 준비
dimension_df = spark.read.csv(
    "file:///workspace/data/ecommerce_region.csv",
    header=True,
    inferSchema=True
)

# 6. 조인 쿼리 작성 (DPP 발생 지점!)
# 상황: Dimension 테이블에서 'San Francisco'만 골라서 조인합니다.
# Spark는 'San Francisco'의 ID를 알아낸 뒤, read_df의 해당 파티션만 읽습니다.
joined_df = read_df.join(
    F.broadcast(dimension_df),
    read_df.region == dimension_df.region_id,
    "inner"
).where(dimension_df.city == "San Francisco")

joined_df.show()

print("=== Dynamic Partition Pruning Plan ===")
joined_df.explain("formatted")
```

---
## 실행 계획 분석 (`explain`) 

`.explain(mode="formatted")` 로그에서 **DPP가 성공했다는 결정적인 증거(Smoking Gun)** 를 찾아봅시다.

### 확인 포인트 1: `PartitionFilters`

`Scan parquet` 단계의 필터 조건을 확인합니다.

```text
(1) Scan parquet
...
PartitionFilters: [isnotnull(region#120), dynamicpruningexpression(region#120 IN dynamicpruning#173)]
...
```

- **성공 신호:** `dynamicpruningexpression`이 보인다면 DPP가 성공적으로 작동한 것입니다
- **의미:** "실행 중에(`dynamic`) 알아낸 값들(`dynamicpruning#173`)만 읽겠다"는 뜻입니다.

### 확인 포인트 2: `SubqueryBroadcast`

```text
===== Subqueries =====

Subquery:1 Hosting operator id = 1 Hosting Expression = region#120 IN dynamicpruning#173
AdaptiveSparkPlan (9)
+- Filter (8)
   +- Scan csv  (7)
```

- **의미:** Dimension 테이블(지역 정보)에서 필터링된 키 값들이 **Broadcast(방송)** 되어 Fact 테이블 스캔 단계로 넘어갔음을 의미합니다.

> **효과:** `San Francisco`의 ID가 `2`라면, 스파크는 `/tmp/sales_partitioned/region_id=2` 폴더만 읽고, 나머지(`1`, `3`)는 쳐다보지도 않습니다. 
> 이로 인해 I/O가 획기적으로 줄어듭니다.

---
## 핵심 설정 및 제약사항 

DPP는 기본적으로 켜져 있지만, 특정 조건이 맞아야만 발동합니다.

### 설정 (Configuration)

| **설정 키**                                                       | **기본값** | **설명**                             |
| -------------------------------------------------------------- | ------- | ---------------------------------- |
| `{python}spark.sql.optimizer.dynamicPartitionPruning.enabled`  | `true`  | DPP 기능 활성화 여부 (무조건 켜두세요!).         |
| `{python}spark.sql.optimizer.dynamicPartitionPruning.fallback` | `true`  | DPP 적용이 불가능할 경우, 일반 조인으로 자동 전환합니다. |

#### 설정 확인 및 변경 방법 (Python):

```python
# 설정 확인
spark.conf.get("spark.sql.optimizer.dynamicPartitionPruning.enabled")

# 강제로 켜기 (False일 경우)
spark.conf.set("spark.sql.optimizer.dynamicPartitionPruning.enabled", "true")
```

### 제약사항 (Limitations) - "왜 DPP가 안 터지죠?"

다음 3가지 조건 중 하나라도 어긋나면 DPP는 작동하지 않습니다.

1. **Partitioned Table 필수:** 대용량 테이블(Fact Table)은 반드시 물리적으로 **파티셔닝(`partitionBy`)** 되어 있어야 합니다.    
2. **Broadcast 가능 크기:** 필터링된 Dimension 테이블(혹은 키 값들)이 **Broadcast** 될 수 있을 만큼 작아야 합니다. 
	- (설정: `spark.sql.autoBroadcastJoinThreshold` 확인)
3. **조인 유형:** 주로 **Equi-Join (등가 조인, `{text}=` 조건)** 에서만 작동합니다.


---
###  Tip 

"DPP는 **'데이터웨어하우스(DW)'** 스타일 쿼리에서 빛을 발해. 
팩트 테이블(Fact)은 엄청 크고, 디멘전 테이블(Dimension)은 작을 때 말이야! 만약 조인이 너무 느리다면, **큰 테이블이 조인 키 기준으로 파티셔닝(PartitionBy) 되어 있는지**부터 확인해봐. 
그게 안 되어 있으면 DPP는 절대 안 터지니까!"

---
## [Tip] `.explain()` 모드별 활용법 💡

"로그가 너무 복잡해요! 어떤 걸 써야 하나요?"

|**상황**|**추천 명령어**|**이유**|
|---|---|---|
|**"DPP 적용됐는지 확인하고 싶어"**|`.explain(mode="formatted")`|`PartitionFilters` 항목을 깔끔하게 보여주니까요.|
|**"Broadcast Join인지 보고 싶어"**|`.explain(mode="formatted")`|트리 구조에서 `BroadcastHashJoin` 노드가 바로 보입니다.|
|**"스파크가 내 쿼리를 어떻게 바꿨는지 궁금해"**|`.explain(True)`|`Optimized Logical Plan`과 `Physical Plan`을 비교해볼 수 있습니다.|
