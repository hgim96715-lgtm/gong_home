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
##  미스터리: "왜 폴더로 저장되나요?" 

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
- 스파크는 **분산 처리 시스템**입니다.
- 작업자(Executor) 100명이 동시에 데이터를 저장하는데, 파일 하나에 몰아 쓰려고 하면 **병목 현상(줄 서기)** 이 생깁니다.
- 그래서 **"일단 각자 알아서 파일로 저장해! 폴더에 모아두기만 해!"** 방식(Distributed Write)을 씁니다.

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

|**모드 (Mode)**|**설명**|**위험도**|
|---|---|---|
|**`error`** (기본값)|폴더가 있으면 **에러 내고 멈춤**. (안전 제일)|🟢 안전|
|**`overwrite`**|기존 폴더를 **싹 지우고** 새로 만듦.|🔴 위험 (덮어쓰기)|
|**`append`**|기존 폴더 안에 **새 파일들을 추가**함.|🟡 주의 (중복 데이터 가능성)|
|**`ignore`**|있으면 그냥 **무시**하고 아무것도 안 함.|⚪️ 글쎄...|


---
## 실전 코드 패턴 (Best Practice) 

가장 많이 쓰는 포맷은 CSV가 아니라 **Parquet(파케이)** 입니다.

```python
# 1. 쓰기 (Write)
df.write\
    .mode("overwrite")\       # 덮어쓰기 허용
    .option("header", True)\  # (CSV일 때만) 첫 줄에 컬럼명 포함
    .csv("my_data_folder")    # 폴더 이름 지정

# 2. 읽기 (Read)
# 폴더 경로만 주면 알아서 안의 part 파일들을 다 읽어들임
df_load = spark.read\
    .option("header", True)\
    .csv("my_data_folder")
```


