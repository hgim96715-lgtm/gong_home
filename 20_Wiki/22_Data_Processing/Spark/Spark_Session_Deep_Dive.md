---
aliases:
  - SparkSession심화
  - SparkConf
  - 스파크설정튜닝
  - KryoSerializer
  - ShufflePartitions
tags:
  - PySpark
  - Spark
related:
  - "[[PySpark_Session_Context]]"
  - "[[Spark_Architecture]]"
---
##  SparkSession이란? (The Unified Entry Point)

과거(Spark 1.x)에는 `SparkContext`(Core), `SQLContext`(SQL), `HiveContext`(Hive)가 따로 놀았습니다.
Spark 2.0부터는 **`SparkSession`** 하나로 이 모든 기능을 통합했습니다. 
스파크를 켠다는 것은 곧 **"세션을 맺는다"** 는 뜻입니다.

---
## Builder Pattern 해부 

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("DeepDiveApp") \
    .config("spark.some.config.option", "some-value") \
    .enableHiveSupport() \
    .getOrCreate()
```

### ① `.master(url)`

- **역할:** "누구한테 일을 시킬까?" (Cluster Manager 지정)
    
- **옵션:**
    - `local`: 싱글 스레드 (비추천).
    - `local[*]`: **[로컬 실습용]** 가용 가능한 모든 코어 사용.
    - `local[4]`: 코어 4개만 사용.
    - `spark://host:port`: Standalone 클러스터 Master 주소.
    - `yarn`: 하둡 YARN 클러스터 (현업 표준).
    - `k8s://...`: 쿠버네티스 클러스터.

### ② `.appName(name)`

- **역할:** "이 작업 이름이 뭐야?"
- **중요성:** Spark UI(`4040` 포트)나 YARN Resource Manager 화면에 이 이름이 뜹니다. 
- 이름을 대충 지으면 나중에 로그 찾을 때 고생합니다.

### ③ `.enableHiveSupport()`

- **역할:** 스파크가 하둡의 **Hive 메타스토어**와 연결할 수 있게 해줍니다.
- **사용 시기:** Hive 테이블을 읽거나 써야 할 때 필수입니다. (현업 데이터 엔지니어링에선 거의 기본값)

### ④ `.getOrCreate()` ️

- **역할:** **싱글톤(Singleton)** 패턴 유지.
- **설명:** 이미 실행 중인 SparkSession이 있으면 그것을 반환하고, 없으면 새로 만듭니다.
- **주의:** 하나의 JVM(Java Virtual Machine) 안에는 **단 하나의 SparkContext**만 존재할 수 있습니다. 
	 그래서 무조건 `new` 하지 않고 `getOrCreate`를 씁니다.

---
## 핵심 `.config()` 설정 Top 5

`.config("Key", "Value")`로 수천 가지 옵션을 제어합니다. 
데이터 엔지니어가 반드시 알아야 할 5가지입니다.

### 셔플 파티션 개수 (성능 튜닝 1순위)

```python
.config("spark.sql.shuffle.partitions", "200")  # 기본값
```

- **의미:** `join`이나 `groupBy`를 할 때 데이터를 몇 개의 조각(Partition)으로 쪼개서 섞을 것인가?
- **문제점:** 기본값 200은 로컬 실습이나 작은 데이터(MB 단위)에는 너무 큽니다. 오버헤드 발생.
    
- **추천:**
    - **로컬 실습:** `"5"` ~ `"10"`
    - **대용량(TB) 처리:** `"500"` ~ `"2000"`

### 메모리 설정 (OOM 방지)

```python
.config("spark.driver.memory", "2g")    # 반장 머리 크기
.config("spark.executor.memory", "4g")  # 일꾼 힘
```

- **Driver Memory:** `collect()` 등으로 결과를 가져올 때 버티는 힘. 부족하면 OOM(Out Of Memory).
- **Executor Memory:** 실제 데이터 연산(`map`, `filter`)을 버티는 힘.

### 직렬화 (Serialization) 속도 향상

```python
.config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

- **의미:** 데이터를 네트워크로 보낼 때 압축하는 방식.
- **효과:** 기본 `JavaSerializer`보다 **KryoSerializer**가 훨씬 빠르고 용량도 적게 차지합니다. (거의 필수 설정)

### UI 포트 변경 (충돌 방지)

```python
.config("spark.ui.port", "4041")
```

### 동적 할당 (Dynamic Allocation)

```python
.config("spark.dynamicAllocation.enabled", "true")
```

- **의미:** "일 없으면 Executor 퇴근시키고, 일 많으면 알바생 더 불러와."
- **효과:** 클러스터 자원을 효율적으로 씁니다. (YARN 환경에서 주로 사용)

---
## 세션 종료 (Housekeeping) 

작업이 끝나면 반드시 세션을 닫아줘야 자원(CPU, Memory)을 반납합니다.

```python
# 작업 끝!
spark.stop()
```

- Jupyter Notebook에서는 커널을 재시작(Restart Kernel)하면 자동으로 `stop()` 되지만,
- 스크립트(.py) 파일에서는 명시적으로 적어주는 것이 예의입니다.