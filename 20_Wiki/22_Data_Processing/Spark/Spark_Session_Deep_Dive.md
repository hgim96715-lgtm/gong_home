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
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약 

"스파크를 켜는 방법은 두 가지가 있다: 코드로 켤 때(`SparkSession`)와 실행할 때(`spark-submit`)."
- **SparkSession:** 코드 안에서 스파크의 모든 기능(SQL, Hive 등)을 사용하는 **통합 출입문**입니다.
- **spark-submit:** 작성한 코드를 실제 서버(클러스터)에 던져서 실행시키는 **배포 명령어**입니다.

---
## 코드에서의 시작: `SparkSession` 

스파크 2.0부터는 `SparkContext`, `SQLContext`, `HiveContext`가 모두 이 하나로 통합되었습니다.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("local[*]") \
    .appName("DeepDiveApp") \
    .config("spark.sql.shuffle.partitions", "200") \
    .enableHiveSupport() \
    .getOrCreate()
```

### 주요 메서드 해부

|**메서드**|**설명**|
|---|---|
|**`.master(url)`**|"누구한테 일을 시킬까?" (Cluster Manager). 실무에선 보통 비워두고 `spark-submit`에서 지정합니다.|
|**`.appName(name)`**|Spark UI나 YARN 목록에 뜰 이름표입니다. (로그 찾을 때 중요!)|
|**`.enableHiveSupport()`**|**필수.** 하둡 Hive 메타스토어와 연결하여 테이블을 읽고 쓸 수 있게 합니다.|
|**`.getOrCreate()`**|**싱글톤 유지.** 이미 켜진 세션이 있으면 그걸 쓰고, 없으면 새로 만듭니다.|

---
## 실행에서의 시작: `spark-submit` 

코드를 다 짰으면, 실제 클러스터에 자원을 요청하며 실행해야 합니다. 이때 쓰는 명령어가 `spark-submit`입니다.

### 기본문법

```bash
./bin/spark-submit \
	--class <메인 클래스> \ 
	--master <마스터 URL> \
	--deploy-mode <모드> \
	--conf <키>=<값> \
	... # 기타 옵션들
	<애플리케이션 파일 경로> \
	[애플리케이션 인자]
```

**필수 옵션 예시 (PySpark)**

```bash
./bin/spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --driver-memory 8g \
  --executor-memory 16g \
  --executor-cores 2 \
  wordByExample.py
```

### ① 핵심 옵션 1: Master (어디서 돌릴까?)

| **옵션 값**                | **설명 & 사용법**                                                                                                                                              |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`local[k]`**          | **로컬 모드**<br><br>• `k`개의 워커 스레드로 내 컴퓨터에서 실행합니다.<br><br>• `local[*]`: 가용 가능한 모든 코어 사용.                                                                     |
| **`yarn`**              | **YARN 클러스터 (실무 표준)**<br><br>• 하둡 YARN에게 자원 관리를 맡깁니다.<br><br>• 현업에서 가장 많이 사용되는 방식입니다.                                                                     |
| **`spark://HOST:PORT`** | **Standalone 클러스터**<br><br>• 스파크 자체 클러스터 매니저를 사용할 때 씁니다.<br><br>• `HOST`와 `PORT`를 실제 마스터 노드의 주소로 바꿔서 입력해야 합니다.                                            |
| **`k8s://HOST:PORT`**   | **쿠버네티스(Kubernetes) 클러스터**<br><br>• `k8s://` 뒤에 쿠버네티스 마스터의 주소와 포트를 적습니다.<br><br>• 기본적으로 `https` 연결을 사용하며, 보안 연결을 명시하려면 `k8s://https://HOST:PORT` 형식을 씁니다. |


### ② 핵심 옵션 2: Deploy Mode (드라이버 위치)

스파크의 두뇌인 **Driver(드라이버)** 를 어디에 띄울지 결정합니다.

#### **Client Mode (`--deploy-mode client`)**

- **특징:** Driver가 **명령어를 실행한 내 컴퓨터(Client)** 에서 실행됩니다.
- **용도:** 결과 로그를 바로 봐야 하는 **대화형(Interactive)** 작업이나 **디버깅** 용도입니다.
- **주의:** 내 컴퓨터를 끄면 앱도 죽습니다.

#### **Cluster Mode (`--deploy-mode cluster`)**

- **특징:** Driver가 **클러스터 내부의 워커 노드(Worker Node)** 중 한 곳에서 실행됩니다.
- **용도:** **운영(Production) 배포용**입니다.
- **장점:** 내 컴퓨터를 꺼도 클러스터 안에서 알아서 잘 돌아갑니다.

### ③ 자원 할당 (Resources)

성능과 직결되는 옵션입니다.

- **`--driver-memory`**: 드라이버가 사용할 메모리 (예: `8g`)
- **`--executor-memory`**: 각 익스큐터가 사용할 메모리 (예: `16g`)
- **`--executor-cores`**: 각 익스큐터가 사용할 CPU 코어 수 (예: `2`)

### ④ PySpark 전용 설정 (Dependency & Env)

파이썬 파일을 배포할 때는 추가 설정이 필요합니다.

- **`--py-files`**: `.py`, `.zip`, `.egg` 등 의존성 파일을 함께 전송합니다. (이거 안 보내면 `ModuleNotFoundError` 남!)
	-  예: `spark-submit --py-files file1.py,file2.zip wordByExample.py`
- **환경 변수 설정**: Driver와 Executor의 파이썬 버전을 맞출 때 사용합니다.
	- `export PYSPARK_PYTHON=/usr/bin/python3`

----
## 코드 vs 명령어: 누가 이기나? 

**"코드(`SparkSession`)에도 적고 명령어(`spark-submit`)에도 적으면 누가 이길까요?"**

1. **SparkConf (코드 내부):** 가장 기본 설정입니다.
2. **Spark-Submit (명령어):** **코드를 덮어씁니다 (Override).** (우선순위 높음)
    - 코드에 `.master("local")`이라고 적었어도,
    - `spark-submit --master yarn`으로 실행하면 **YARN**에서 돕니다.

> **💡 Best Practice:** 환경에 따라 바뀌는 값(Master, Memory 등)은 코드에 하드코딩하지 말고, **`spark-submit` 옵션으로 외부에서 주입**하세요!


---
## 데이터 엔지니어 필수 설정 Top 5 (`.config`) 

코드 내에서 `.config("key", "value")`로 제어해야 하는 성능 옵션들입니다.

### ① 셔플 파티션 개수 (성능 튜닝 1순위)

```python
spark.conf.set("spark.sql.shuffle.partitions", "200") # 기본값 200
```

- `Join`이나 `GroupBy` 시 데이터를 몇 조각으로 쪼갤지 결정합니다.
- **로컬 실습:** `"5"`~`"10"` (추천) / **대용량:** `"500"`~`"2000"`

### ② 메모리 설정 (OOM 방지)

```python
# 보통 spark-submit 인자로 넘기는 것이 좋음
.config("spark.driver.memory", "2g")    # 반장(Driver)
.config("spark.executor.memory", "4g")  # 일꾼(Executor)
```

### ③ 직렬화 (Serialization) 속도 향상

```python
.config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

- 기본 `JavaSerializer`보다 훨씬 빠르고 가벼운 **`Kryo`** 방식을 사용합니다.

### ④ 포트 충돌 방지

```python
.config("spark.ui.port", "4041")
```

- 한 서버에서 여러 스파크 앱을 띄울 때 포트(4040)가 겹치지 않게 합니다.

### ⑤ 동적 할당 (Dynamic Allocation)

```python
.config("spark.dynamicAllocation.enabled", "true")
```

- 일의 양에 따라 Executor 개수를 자동으로 늘렸다 줄였다 합니다. (YARN 필수 설정)


---
## 마무리 (Housekeeping) 

작업이 끝나면 반드시 세션을 닫아 자원을 반납해야 합니다.

```python
spark.stop()
```

