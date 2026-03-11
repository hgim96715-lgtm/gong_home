---
aliases:
  - createOrReplaceTempView
  - spark.sql
  - GlobalTempView
  - TempView
  - 스파크SQL사용법
tags:
  - SQL
  - Spark
related:
  - "[[Spark_DataFrame_SQL_Intro]]"
  - "[[PySpark_Session_Context]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[SQL_Concepts_View]]"
  - "[[Spark_DataFrame]]"
---
## 개념 한 줄 요약 

**"나의 DataFrame에 이름을 붙여서, SQL 세상에서도 부를 수 있게 만드는 등록 절차."**

* **`DataFrame`**: 파이썬 코드(`df.select...`)로 제어함.
* **`TempView`**: SQL 코드(`SELECT * FROM ...`)로 제어함.
* **`createOrReplaceTempView`**: "이 데이터프레임을 이제부터 'people'이라는 테이블 이름으로 부를게!" 라고 스파크에게 알려주는 함수.

---
## 핵심 함수: `createOrReplaceTempView` 

가장 많이 쓰는 함수입니다. 
이름 그대로 **"임시 뷰를 만들거나(Create), 이미 있으면 교체(Replace)해라"** 라는 뜻입니다.
SQL을 쓰기 위해 '테이블 이름' 등록 (이게 없으면 SQL 못 씀!)

### ① 기본 사용법

```python
# 1. 데이터프레임 생성
df = spark.read.json("people.json")

# 2. SQL을 쓰기 위해 '테이블 이름' 등록 (이게 없으면 SQL 못 씀!)
# "df야, 너 이제부터 SQL 세계에서는 'my_table'이라고 불릴 거야."
df.createOrReplaceTempView("my_table")

# 3. 이제 SQL 마음껏 사용 가능
sql_df = spark.sql("SELECT * FROM my_table WHERE age > 20")
sql_df.show()
```

### ② 특징 (Session Scope) ⏳

- **임시(Temp)인 이유:** 이 테이블은 **현재 연결된 세션(`spark`)이 살아있는 동안에만** 유효합니다.
- **소멸:** `spark.stop()`을 하거나 노트북을 끄면 **사라집니다.** (DB에 영구 저장되는 게 아님!)
- **범위:** 내가 켠 노트북 안에서만 보입니다. (다른 사람이 켠 노트북에서는 안 보임)

> **참고:** 예전 코드에서는 `.registerTempTable()`을 썼지만, 지금은 deprecated(사용 권장 안 함) 되었습니다. 
> 무조건 `createOrReplaceTempView`를 쓰세요.

---
## [심화] 전역으로 쓰기: `createGlobalTempView` 

만약 **"다른 노트북(세션)에서도 이 테이블을 공유해서 쓰고 싶다"** 면 이걸 씁니다.

- **범위:** 하나의 스파크 애플리케이션 전체에서 공유됨.
- **접근법:** 이름 앞에 반드시 **`global_temp.`** 를 붙여야 함.

```python
# 1. 전역 뷰 등록
df.createGlobalTempView("shared_table")

# 2. 접근할 때 (global_temp 데이터베이스 밑에 있음)
spark.sql("SELECT * FROM global_temp.shared_table").show()
```

---
## [개념 정리] TempView vs GlobalTempView

|**구분**|**createTempView**|**createGlobalTempView**|
|---|---|---|
|**비유**|**내 책상 위의 메모지**|**거실 벽에 붙인 공지사항**|
|**범위**|현재 **SparkSession** 안에서만 보임.|스파크 애플리케이션 전체에서 공유됨.|
|**저장 위치**|현재 데이터베이스 (보통 `default`)|**`global_temp`** 라는 시스템 DB|
|**목록 확인**|`spark.catalog.listTables()`|`spark.catalog.listTables("global_temp")`|
|**SQL 사용법**|`SELECT * FROM income`|`SELECT * FROM global_temp.income` (접두어 필수!)|
|**수명**|세션(`spark.stop()`) 끝나면 사라짐|애플리케이션 종료 시 사라짐|


---
## 테이블 관리하기 (Catalog) 

등록된 테이블이 뭐가 있는지 확인하거나 지울 때 사용합니다.

```python
# 1. 테이블 목록 확인
spark.catalog.listTables()

# 2. 뷰 삭제 (메모리 정리)
spark.catalog.dropTempView("my_table")
```

---
## 실전 패턴: Python과 SQL의 티키타카 

현업에서는 파이썬과 SQL을 섞어서 쓰는 게 국룰입니다.

```python
# Step 1. 복잡한 로직은 Python 함수로 처리 (UDF 등)
df = spark.read.csv("data.csv")
df_clean = df.filter(...) 

# Step 2. 집계나 조인은 SQL이 편하니까 변환!
df_clean.createOrReplaceTempView("clean_data")

# Step 3. SQL로 복잡한 쿼리 작성
result_df = spark.sql("""
    SELECT category, count(*) as cnt
    FROM clean_data
    GROUP BY category
    HAVING cnt > 100
""")

# Step 4. 결과는 다시 DataFrame이므로 파이썬으로 마무리
result_df.write.parquet("final_result")
```

---
### 요약

"**`createOrReplaceTempView`** 는 **파이썬 구역**에 있는 데이터를 **SQL 구역**으로 넘겨주는 **'통행증 발급소'** 라고 생각하면 돼. 
이걸 안 하면 `spark.sql("SELECT ...")`을 썼을 때 **'그런 테이블 없는데요?'** 하고 에러가 나니까, SQL을 쓰기 전엔 무조건 필수야!"


