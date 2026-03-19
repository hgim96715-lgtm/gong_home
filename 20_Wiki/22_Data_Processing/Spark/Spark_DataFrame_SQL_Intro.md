---
aliases:
  - DataFrame
  - SparkSQL
  - CatalystOptimizer
  - 데이터프레임
  - 스파크SQL
tags:
  - Spark
  - SQL
  - DataFrame
related:
  - "[[RDD_Concept]]"
  - "[[Spark_Session_Context]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[SQL_with_Spark]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step5.ipynb
---
## Spark DataFrame이란?

**"RDD에 '스키마(Schema)'라는 이름표를 붙여서 테이블처럼 만든 것."**

* **정의:** 행(Row)과 열(Column)로 구성된 **분산 데이터 컬렉션**입니다
* **비유:**
    * **RDD:** 내용물을 알 수 없는 텍스트 덩어리가 든 자루. (열어봐야 암)
    * **DataFrame:** 엑셀 시트나 RDBMS 테이블. (제목과 데이터 타입이 딱 정해져 있음) 

### 🌟 핵심 특징

1.  **스키마 지원 (Schema Support):** 각 컬럼의 이름과 데이터 타입(String, Int 등)이 명확히 정의되어 있어 처리가 효율적입니다
2.  **게으른 연산 (Lazy Evaluation):** RDD와 마찬가지로, 액션(Action)이 호출되기 전까지는 실행 계획만 세우고 기다립니다
3.  **최적화 (Optimization):** **Catalyst Optimizer**라는 똑똑한 엔진이 쿼리 실행 계획을 자동으로 최적화해줍니다 (RDD에는 없음!)
4.  **분산 처리:** 클러스터 노드들에 데이터가 분산되어 병렬로 처리됩니다

---
##  RDD vs DataFrame 완벽 비교 

왜 다들 RDD 안 쓰고 DataFrame을 쓸까요?

| 구분 | **DataFrame (신식) 🚅** | **RDD (구식) 🐢** |
| :--- | :--- | :--- |
| **추상화 수준** | **High-level** (구조화된 테이블 형태) | **Low-level** (키-값 쌍, 날것의 객체) |
| **사용 편의성** | **쉬움** (SQL처럼 직관적 API) | **어려움** (복잡한 람다 함수 사용) |
| **성능** | **최적화됨** (Catalyst Optimizer가 도와줌) | **최적화 없음** (사용자가 직접 튜닝해야 함) |
| **스키마** | **있음 (Yes)** | **없음 (No)** |
| **주 용도** | 정형/반정형 데이터 (CSV, JSON, Parquet) | 비정형 데이터, 아주 복잡한 커스텀 로직 |

---
## SparkSQL이란? 

**"스파크 데이터를 SQL 문법으로 다루게 해주는 모듈."**

* **정의:** 구조화된 데이터를 처리하기 위한 스파크 모듈로, SQL과 유사한 문법을 제공합니다
* **특징:**
    * SQL 쿼리를 작성하면 스파크가 알아서 분산 처리를 수행합니다
    * **Hive**와 연동하여 기존 하둡 데이터도 쉽게 조회할 수 있습니다
    * **BI 도구(Tableau 등)** 와 JDBC/ODBC로 연결하여 대시보드를 만들 수 있습니다

###  왜 SparkSQL을 쓰나요?

1.  **쉬운 사용성:** 복잡한 파이썬/스칼라 코드를 몰라도, **SQL만 알면** 빅데이터 처리가 가능합니다
2.  **통합성:** SQL로 쿼리한 결과를 다시 DataFrame으로 변환해서 파이썬 코드로 가공할 수 있습니다 (왔다 갔다 가능)
3.  **성능:** SQL 쿼리도 결국 **Catalyst Optimizer**와 **Tungsten 엔진**을 거치므로 엄청나게 빠릅니다

---
## 실전 사용 패턴 (Code Snippet)

SQL과 DataFrame은 물 흐르듯이 섞어 쓸 수 있습니다.

### ① SQL 스타일

```python
# 데이터를 테이블처럼 등록
df.createOrReplaceTempView("people")

# SQL 쿼리 실행
results = spark.sql("SELECT * FROM people WHERE age > 20")
```

### ② DataFrame 스타일 (DSL)

```python
# 함수 체이닝 방식
results = df.select("*").filter(df.age > 20)
```

**결론:** 둘 중 편한 걸 쓰면 됩니다. 내부적으로는 **똑같은 엔진**이 돌아갑니다!