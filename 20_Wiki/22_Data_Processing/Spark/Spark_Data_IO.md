---
aliases:
  - SparkIO
  - read
  - write
  - save
  - load
  - parquet
  - csv
  - sveMode
tags:
  - Spark
related:
  - "[[Spark_DataFrame_Basics]]"
  - "[[Spark_Graph_Analysis_Basic]]"
  - "[[00_Apache_Spark_HomePage]]"
---
##  개념 한 줄 요약 

**"스파크의 저장(Write)은 '파일 하나'를 만드는 게 아니라, '결과물 폴더'를 만들고 그 안에 여러 작업자가 나눠서 저장하는 것이다."**

* **Read:** `spark.read...` (데이터 불러오기)
* **Write:** `df.write...` (데이터 내보내기)

---
##  "왜 폴더로 저장되나요?" 

파이썬 `pandas`는 `result.csv` 파일 하나를 딱 뱉어내지만, 스파크는 다릅니다.

```python
# "output"이라는 이름으로 저장해줘!
df.write.csv("output")
```

**실제 저장된 모습 (폴더 구조)**

```scss
output/               <-- 1. 네가 지정한 이름은 '폴더'가 됨
├── _SUCCESS          <-- 2. "저장 성공했음!" 도장 (빈 파일)
├── part-00000.csv    <-- 3. 작업자 1이 저장한 조각
├── part-00001.csv    <-- 4. 작업자 2가 저장한 조각
└── part-00002.csv    <-- 5. 작업자 3이 저장한 조각
```

**이유 (Why)**
Spark는 **분산 처리 시스템**이다. 
작업자(Executor) 수백 명이 동시에 저장하는데, 파일 하나에 몰아쓰면 병목이 생긴다. 
그래서 **"각자 알아서 파일로 저장하고, 폴더에 모아두기만 해!"** 방식(Distributed Write)을 쓴다.

>**💡 꿀팁:** 파일 하나로 깔끔하게 보고 싶다면? 
>저장하기 직전에 **`coalesce(1)`** 을 써서 작업자를 1명으로 줄여야 합니다. 
>(단, 병렬 처리를 포기하는 것이라 속도는 느려집니다.)
>**[[Spark_Graph_Analysis_Basic#Step 2 하나의 파일로 모아서 저장 (Serialization & Export)|👉 coalesce(1) 실전 사용법 보러가기]]**

```python
data.coalesce(1).write.csv("output")
```

---
## 저장 모드 (Save Mode) 

"이미 폴더가 있는데 또 저장하면 어떻게 되나요?" -> **에러 납니다!** 
그래서 `mode` 옵션을 잘 써야 합니다.

- 문법 : `{python}df.write.mode("옵션").csv("폴더명")`

```python
# 문법: df.write.mode("옵션").csv("폴더명")

df.write.mode("overwrite").csv("output")
```

|**모드 이름**|**코드**|**실제 동작**|**추천 상황**|
|---|---|---|---|
|**Error**|`.mode("error")`|이미 있으면 **중단(에러)**|데이터 소실 방지 (기본값)|
|**Overwrite**|**`.mode("overwrite")`**|기존꺼 **다 지우고** 새로 생성|일일 리포트 갱신|
|**Append**|**`.mode("append")`**|기존꺼 뒤에 **이어서 추가**|매시간 쌓이는 로그 데이터|
|**Ignore**|`.mode("ignore")`|이미 있으면 **아무것도 안 함**|중복 실행 방지|

---
## 실전 코드 패턴 (Best Practice) 

### 파일로 저장하기 (File Write)

실무에서는 CSV보다 **Parquet(파케이)** 포맷을 더 많이 쓴다.
Spark는 분산 처리 엔진이므로, 파일을 저장할 때 **'폴더'**를 만들고 그 안에 여러 작업자가 각자 조각 파일(`part-xxxx`)을 뱉어냅니다.

```python
# 1. 쓰기 (Write)
df.write \
    .format("parquet")      # 포맷 지정 (parquet / csv / json 등)
    .mode("overwrite") \    # 덮어쓰기 허용
    .save("output_folder")  # 폴더 이름
```

#### CSV를 써야 한다면 `header` 옵션을 챙길 것.

```python
df.write \
    .format("csv") \
    .mode("overwrite") \
    .option("header", True) \   # 첫 줄에 컬럼명 포함
    .save("output_folder")
```

### 데이터베이스로 저장하기 (JDBC Write)

```python
df.write.jdbc(
    url="jdbc:postgresql://postgres:5432/airflow", # 1. 도착지 주소 (내부망)
    table="report_category_sales",                 # 2. 만들거나 채울 테이블 이름
    mode="overwrite",                              # 3. 기존 리포트 삭제 후 생성
    properties={                                   # 4. 통역사 및 인증정보
        "user": "airflow",
        "password": "airflow",
        "driver": "org.postgresql.Driver"
    }
)
```

- **`url`** → 어디로 쏠까? 도커 내부망은 `jdbc:postgresql://postgres:5432/DB명` 형식
- **`table`** → 어떤 이름으로? 테이블이 없으면 Spark가 스키마 보고 **자동 생성**
	- `overwrite` : 기존 테이블 DROP 후 재생성 → 리포트 갱신용
	- `append` : 기존 데이터 아래에 덧붙임 → 로그 누적용
- **`properties`** → 인증 정보 + 드라이버. `driver` 키를 **반드시** 명시해야 한다.
