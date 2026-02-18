---
aliases:
  - SparkSession
  - SparkContext
  - sc
  - spark
  - RDD생성
  - 세션설정
  - parallelize
  - textFile
tags:
  - Spark
related:
  - "[[RDD_Concept]]"
  - "[[Spark_Installation_Local]]"
  - "[[Spark_Architecture]]"
  - "[[Spark_Session_Deep_Dive]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Transformations_vs_Actions]]"
  - "[[00_Apache_Spark_HomePage]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step1.ipynb
---
## 개념 한 줄 요약 

**"스파크 클러스터로 들어가는 '문(Door)'을 여는 객체."**

* **`SparkContext` (`sc`):** (구식/엔진) RDD를 다루기 위한 원조 연결 통로. 엔진의 저수준 기능 제어.
* **`SparkSession` (`spark`):** (신식/자동차) RDD뿐만 아니라 DataFrame, SQL까지 모두 다루는 **통합 관리자**. (내부에 `sc`를 포함함)

---
##  `spark` vs `sc` 완벽 비교 (이걸 알아야 안 헷갈림) 

| 구분 | **spark (SparkSession)** | **sc (SparkContext)** |
| :--- | :--- | :--- |
| **비유** | **사장님 (자동차)** 🚘 | **공장장 (엔진)** ⚙️ |
| **주 역할** | **DataFrame** 만들기 (SQL, 테이블) | **RDD** 만들기 (리스트, 비정형) |
| **상하관계** | 전체 관리자 (Top-Level) | 실무 처리자 (Sub-Level) |
| **코드 패턴** | `spark.read.csv(...)` | `sc.parallelize(...)` |
| **결론** | **평소엔 `spark`만 쓴다.** | **`parallelize` 쓸 때만 `sc`를 꺼낸다.** |

---
##  현대적인 방식: SparkSession (권장 )

현업에서는 **Builder 패턴**으로 안전하게 세션을 만듭니다.

```python
from pyspark.sql import SparkSession

# 1. SparkSession 만들기 (없으면 만들고, 있으면 가져와라)
spark = (SparkSession.builder
    .master("spark://spark-master:7077") \
    .appName("MyFirstSparkApp") \
    .config("spark.driver.memory", "2g") \
    .getOrCreate())

# 2. 숨겨진 SparkContext 꺼내기 (엔진룸 열기)
# (RDD를 쓰고 싶으면 session 안에서 꺼내서 씁니다)
sc = spark.sparkContext
```

> **참고:** 주요 파라미터(`master`, `appName` 등)에 대한 자세한 해석은 **[[Spark_Session_Deep_Dive]]** 를 참고하세요.

---
## 코드 생성 확인

코드를 실행한 뒤, 아래 두 줄을 꼭 쳐보세요. 객체 정보가 뜨면 성공입니다.

```python
# 1. 세션 객체 확인 (메모리 주소가 나와야 함)
print(spark)
# 출력 예: <pyspark.sql.session.SparkSession object at 0x7f...>

# 2. 스파크 버전 확인 (설치된 버전이 맞는지)
print(spark.version)
# 출력 예: 3.5.1
```

---
## 구체적으로 언제 `sc`가 필요한가요?

Dataframe(`spark`)으로는 못 하고, **꼭 `sc`가 있어야만 할 수 있는 작업** 3가지가 있습니다.

#### ① 리스트를 RDD로 만들 때 (`parallelize`)

우리가 연습할 때 가장 많이 쓰는 함수죠? 이 함수는 `spark`에는 없고 `sc`에만 있습니다.

```python
data = [1, 2, 3, 4, 5]

# [X] 에러! spark에는 parallelize 기능이 없음
# rdd = spark.parallelize(data) 

# [O] sc를 꺼내야만 사용 가능
sc = spark.sparkContext
rdd = sc.parallelize(data)
```

#### ② 텍스트 파일을 '날 것' 그대로 읽을 때 (`textFile`)

`spark.read.text()`는 DataFrame(행/열 구조)으로 가져오지만, 그냥 **줄글(String) 덩어리 RDD**로 가져오고 싶으면 `sc`를 써야 합니다.

```python
# RDD로 읽기 (로그 파일 등 비정형 데이터 처리할 때 유리)
rdd = sc.textFile("data.txt")
```

#### ③ 고급 기능 (브로드캐스트 변수, 어큐뮬레이터)

 **"모든 서버에 변수 뿌리기(Broadcast)"** 기능은 `sc`가 관리합니다.

---
## 구식 방식: SparkContext 직접 생성 (Legacy) 

오래된 코드나 인터넷 예제를 보면 `SparkSession` 없이 바로 `sc`를 만드는 경우가 있습니다. 
틀린 건 아니지만, **SQL이나 DataFrame을 못 쓰는 제약**이 있습니다.

```python
import pyspark

# sc 객체 직접 생성 (클러스터 연결)
# 이렇게 하면 'spark' 객체(DataFrame 기능)는 못 씀!
sc = pyspark.SparkContext('local[*]')
```

> **주의:** 한 프로그램 안에서 `SparkContext`는 딱 하나만 존재할 수 있습니다.
>  위처럼 `sc`를 직접 만들었다면 `SparkSession`을 또 만들 때 충돌이 날 수 있으니 주의해야 합니다.

---
## [실습] 연습 해보기 - `step1.ipynb`

```python
# 1. 세션 생성 (가장 안전한 방법)
from pyspark.sql import SparkSession

spark = (
    SparkSession
    .builder
    .appName("RDD-Practice")
    .master("spark://spark-master:7077") # 로컬이라면 "local[*]"
    .getOrCreate()
)

# 2. Context 가져오기 (RDD를 만들기 위해 엔진 꺼냄)
sc = spark.sparkContext

# 3. 데이터 생성 (0부터 9999까지 리스트 -> RDD 변환)
# [중요] parallelize는 sc에만 있다!
big_rdd = sc.parallelize(range(10000))

# 4. 변환 (짝수만 남기기 - Transformation) 
even_rdd = big_rdd.filter(lambda x: x % 2 == 0)

# 5. 행동 (개수 세기 - Action) 
count = even_rdd.count()

print(f"짝수의 개수: {count}")
print(f"샘플 데이터: {even_rdd.takeSample(False, 5)}")

# 6. 종료 (자원 반납 - 필수!)
spark.stop()
```

---
### [트러블 슈팅] 실행할 때 경고가 떠요!

실행 시 빨간색으로 **`WARN NativeCodeLoader...`** 가 뜬다면? 걱정 마세요. 
하둡 압축 라이브러리 관련 경고인데 실습엔 지장이 없습니다. 
자세한 내용은 **[[WARN_NativeCodeLoader_Log]]** 를 참고하세요.