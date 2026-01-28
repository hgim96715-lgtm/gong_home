---
aliases:
  - createDataFrame
  - read
  - show
  - printSchema
  - schema
  - describe
  - 데이터생성
  - 데이터프레임기초
  - toDF
  - withColumnRenamed
tags:
  - Spark
  - DataFrame
  - Basics
  - API
related:
  - "[[Spark_DataFrame_SQL_Intro]]"
  - "[[PySpark_Session_Context]]"
  - "[[Pandas_DataStructures]]"
---
방법 2. 하나씩 콕 집어서 바꾸기 (`withColumnRenamed`)

**"데이터프레임을 만드는 두 가지 방법(파일 읽기 vs 직접 만들기)과 눈으로 확인하는 방법."**

* **Reading:** 외부 파일(CSV, Parquet)을 불러오기. (`spark.read`)
* **Creating:** 리스트나 판다스 데이터를 스파크로 변환하기. (`createDataFrame`)
* **Inspecting:** 잘 만들어졌는지 눈으로 확인하기. (`show`, `printSchema`)

---
## 데이터프레임 만들기 (Manual) 

학습용 데이터나 테스트 데이터를 만들 때 **`createDataFrame`** 을 가장 많이 씁니다.

### ① 기본 문법

```python
# SparkSession.createDataFrame(data, schema=None)
df = spark.createDataFrame(data, schema)
```

### ② 리스트로 만들기 (가장 흔함)

- **바깥쪽 `[...]` (대괄호):** **리스트**입니다. 데이터 전체(테이블)를 의미합니다.
- **안쪽 `(...)` (소괄호):** **튜플**입니다. 데이터 한 행(Row), 즉 한 사람의 정보를 의미합니다.

```python
# 1. 데이터 준비 (리스트 안에 튜플/리스트) 리스트 of 튜플
data = [
    ("James", 30, "M"),
    ("Anna", 25, "F"),
    ("Stark", 40, "M")
]

# 1-2. 리스트 of 리스트 (가능 ) 
data = [["James", 30,"M"], ["Anna", 25,"F"],["Stark",40,"M"]]

#1과 1-2 결과는 똑같은 DataFrame이 나옵니다!

# 2. 컬럼 이름(스키마) 정의
columns = ["Name", "Age", "Gender"]

# 3. 변환!
df = spark.createDataFrame(data, schema=columns)

df.show()
```

### ③ 명시적 스키마 지정 (StructType) 

데이터 타입을 엄격하게 지정하고 싶을 때 사용합니다. (실무 추천)

"스키마(`StructType`)는 여러 개의 컬럼(`StructField`)을 리스트(`[]`)로 묶은 것이다."
- **`StructType`**: **행(Row) 전체**의 구조를 정의하는 틀입니다. 안에는 반드시 **리스트 `[...]`** 가 들어갑니다.
- **`StructField(name, dataType, nullable)`**: **컬럼 하나**의 정의입니다. (이름, 타입, 옵션)
	- **`nullable` (True/False):** **"비어있어도(Null) 됩니까?"**
		- **`True` (기본값/권장):** "네, 값이 없을 수도 있습니다." (안전함)
		- **`False`:** "아니요, 무조건 값이 있어야 합니다!" (데이터에 Null이 있으면 에러 남 )

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

# 스키마를 미리 설계도처럼 만듦
schema = StructType([  # <--- 전체 틀 (리스트로 시작)
    StructField("Name", StringType(), True),  # <--- 1번 컬럼
    StructField("Age", IntegerType(), True), # <--- 2번 컬럼
    StructField("Gender", StringType(), True)
])

df = spark.createDataFrame(data, schema=schema)
```

---
### 자주 쓰는 Data Types 족보

스파크는 파이썬 타입(int, str)을 그대로 쓰지 않고, 전용 타입을 씁니다. 
반드시 `from pyspark.sql.types import *` 로 불러와야 합니다.

|**스파크 타입 (___Type)**|**파이썬 대응**|**설명**|**용도 예시**|
|---|---|---|---|
|**`StringType`**|`str`|문자열|이름, 주소, ID|
|**`IntegerType`**|`int`|정수 (약 ±21억)|나이, 개수, 등수|
|**`LongType`**|`int`|큰 정수 (21억 초과)|유튜브 조회수, 행성 간 거리|
|**`FloatType`**|`float`|실수 (소수점 7자리)|간단한 키, 몸무게|
|**`DoubleType`**|`float`|정밀 실수 (소수점 15자리)|**과학 계산, 정밀한 좌표 (추천)**|
|**`BooleanType`**|`bool`|참/거짓|결혼여부, 성인인증여부|
|**`DateType`**|`datetime.date`|날짜 (연-월-일)|생일, 가입일 (`2024-01-01`)|
|**`TimestampType`**|`datetime.datetime`|날짜 + 시간|로그 발생 시간 (`2024-01-01 12:00:00`)|


---
## 파일에서 불러오기 (Read) 

실무에서는 대부분 파일을 읽습니다. 
**`spark.read`** 객체를 사용합니다.

```python
# 1. CSV 읽기 (헤더가 있고, 스키마를 자동으로 유추해라)
df_csv = spark.read.csv("data.csv", header=True, inferSchema=True)

# 2. JSON 읽기
df_json = spark.read.json("data.json")

# 3. Parquet 읽기 (압축률 좋고 빠름, 스키마 자동 포함)
df_parquet = spark.read.parquet("data.parquet")
```

---
### [팁] CSV 읽을 때 필수 옵션 2가지

```python
spark.read.csv("path", header=True, inferSchema=True)
```

- **`header` (True/False)** : **"첫 번째 줄이 데이터야, 아니면 제목(컬럼명)이야?"** 를 묻는 옵션
    - `True`: 첫 번째 줄은 **제목(Column Name)** 이니까 데이터로 쓰지 말고 컬럼 이름으로 써!"
    - `False`: (기본값): "첫 번째 줄부터 바로 **실제 데이터**가 시작돼. 제목은 없어."(컬럼명은 `_c0`, `_c1`... 자동 생성)
- **`inferSchema` (True/False)**
    - `True`: 스파크가 데이터를 보고 **타입(숫자/문자)을 자동으로 맞춥니다.** (편함, 느림)
    - `False`: 모든 데이터를 **문자열(String)** 로 읽습니다. (불편함, 빠름)

### [꿀팁] 헤더가 없는 파일(`header=False`)에 이름 붙이기

`_c0`, `_c1` 같은 임시 이름이 싫다면 **`.toDF()`** 를 체이닝해서 바로 해결하세요.

**주의:** 컬럼 개수랑 이름 개수가 딱 맞아야 합니다! (컬럼은 3개인데 이름 2개만 주면 에러 남)

```python
# 1. inferSchema로 타입은 자동 추론 
# 2. toDF로 이름은 내가 지정 
df = spark.read.csv("data.csv", header=False, inferSchema=True) \ 
.toDF("Name", "Age", "Job")
```

### 방법 2. 하나씩 콕 집어서 바꾸기 (`withColumnRenamed`)

데이터를 이미 읽어버린 상태에서, 특정 컬럼 이름만 바꾸고 싶을 때 씁니다.

```python
# 일단 읽음 (_c0, _c1 상태)
df = spark.read.csv("...", header=False)

# _c0를 'Price'로 바꿔라
df = df.withColumnRenamed("_c0", "Price") \
       .withColumnRenamed("_c1", "Location")
```

### 방법 3. 아예 처음부터 스키마 지정하기 (`schema`)

아까 배운 `StructType`을 쓰는 정석적인 방법입니다. **이름과 타입을 동시에** 지정합니다.

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

my_schema = StructType([
    StructField("Price", IntegerType(), True),
    StructField("Location", StringType(), True)
])

# header=False여도 내가 schema를 줬으니까 그걸 따름
df = spark.read.csv("...", schema=my_schema, header=False)
```

---
## 데이터 뜯어보기 (Inspection) 

데이터를 만들었으면 확인해야겠죠? 필수 3대장입니다.

### ① `show()`: 데이터 눈으로 보기

```python
# 상위 20줄을 예쁘게 보여줌 (기본값)
df.show()

# 내용이 길어도 자르지 말고 다 보여줘 (truncate=False)
df.show(truncate=False)

# 상위 5줄만 보여줘
df.show(5)
```

### ② `printSchema()`: 뼈대 확인하기

컬럼 이름과 데이터 타입(String, Int 등), Null 허용 여부를 트리 구조로 보여줍니다.

```python
df.printSchema()
# root
#  |-- Name: string (nullable = true)
#  |-- Age: integer (nullable = true)
```

### ③ `describe()`: 기초 통계 확인

숫자 데이터의 개수(count), 평균(mean), 표준편차(stddev), 최소(min), 최대(max)를 계산합니다.

```python
# 1. 모든 컬럼 다 보기 (기본)
df.describe().show()

# 2. 원하는 컬럼만 골라서 보기 (추천 👍)
# 따옴표("")로 감싸서 문자열로 넣어줘야 합니다!
df.describe("members", "rating").show()
```

----
## [팁] `.cache()`는 뭔가요? 

질문하신 코드 `spark.createDataFrame(...).cache()`에서 **`.cache()`** 는 최적화 함수입니다.

- **역할:** "이 데이터프레임은 앞으로 자주 쓸 거니까, **메모리에 박제(Caching)** 해놔!"
- **위치:** 이 내용은 **Level 3 [[Caching_and_Persistence]]** 에서 아주 자세히 다룰 예정입니다. 
- 지금은 **"속도 빠르게 하려고 메모리에 올리는구나"** 정도로만 이해하고 넘어가면 됩니다.





