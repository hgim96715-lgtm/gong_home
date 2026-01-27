---
aliases:
  - SparkSession
  - SparkContext
  - 스파크진입점
  - sc
  - spark
  - RDD생성
tags:
  - PySpark
  - Spark
related:
  - "[[RDD_Concept]]"
  - "[[Spark_Installation_Local]]"
  - "[[Spark_Architecture]]"
  - "[[Spark_Session_Deep_Dive]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Transformations_vs_Actions]]"
---
##  개념 한 줄 요약

**"스파크 클러스터로 들어가는 '문(Door)'을 여는 객체."**

* **SparkContext (`sc`):** (구식) RDD를 다루기 위한 원조 연결 통로. 엔진의 저수준 기능 제어.
* **SparkSession (`spark`):** (신식) RDD뿐만 아니라 DataFrame, SQL까지 모두 다루는 **통합 관리자**. (내부에 SparkContext를 포함하고 있음)
---
## 코드 분석: SparkContext 

 **SparkContext**의 생성자 파라미터들입니다. 하나씩 뜯어봅시다.

```python
import pyspark

# sc 객체 생성 (클러스터 연결)
sc = pyspark.SparkContext('local[*]')
```

---
## 현대적인 방식: SparkSession (Recommended) ️

현업에서는 `SparkContext`를 직접 호출하기보다, **`SparkSession`** 을 빌더 패턴으로 만듭니다. 
이게 훨씬 안전하고 강력합니다.

```python
from pyspark.sql import SparkSession

# 1. SparkSession 만들기 (없으면 만들고, 있으면 가져와라)
spark = SparkSession.builder \
    .master("spark://spark-master:7077") \
    .appName("MyFirstSparkApp") \
    .config("spark.driver.memory", "2g") \
    .getOrCreate()

# 2. 숨겨진 SparkContext 꺼내기
# (RDD를 쓰고 싶으면 session 안에서 꺼내서 씁니다) (필요할 때만)
sc = spark.sparkContext
```

> **Why?** `SparkSession` 하나만 있으면 SQL, Hive, Streaming 기능을 다 쓸 수 있기 때문입니다.
> > 주요 파라미터 해석 은 [[Spark_Session_Deep_Dive|파라미터 깊은해석]] 참조 

### 코드 생성 확인

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
## 연습 해보기 - step1.ipynb

```python
# 1. 세션 생성 (가장 안전한 방법)
from pyspark.sql import SparkSession

spark=(
    SparkSession
    .builder
    .appName("RDD-Practice")
    .master("spark://spark-master:7077")
    .getOrCreate()
)

# 2. Context 가져오기
sc = spark.sparkContext

# 3. 데이터 생성 (0부터 9999까지)
big_rdd = sc.parallelize(range(10000))

# 4. 변환 (짝수만 남기기 - Transformation) -> 게으른 연산이라 즉시 실행 안됨
even_rdd = big_rdd.filter(lambda x: x % 2 == 0)

# 5. 행동 (개수 세기 - Action) -> 이때 실행됨!
count = even_rdd.count()

print(f"짝수의 개수: {count}")
print(f"샘플 데이터: {even_rdd.takeSample(False, 5)}")

# 6. 종료 (자원 반납)
spark.stop()
```

> 이때 실행 해볼때 WARN Native...경고가 뜰수 있다 [[Spark_Installation_Local#① "WARN NativeCodeLoader..." 로그가 떠요!|이게 뭐야!]] 참조 


