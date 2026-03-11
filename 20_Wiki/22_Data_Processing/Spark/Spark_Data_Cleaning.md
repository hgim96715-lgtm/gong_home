---
aliases:
  - Missing Data
  - Null Handling
  - dropna
  - fillna
  - Date Parsing
  - 결측치,
tags:
  - Spark
related:
  - "[[DataFrame_Transform]]"
  - "[[Spark_Functions_Library]]"
  - "[[Pandas_Missing_Values]]"
  - "[[DataFrame_Transform]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Streaming_Window_Aggregation]]"
---
## 결측치 현황 파악하기 (Null Check) 

**"Pandas의 `isna().sum()`이 스파크에는 없습니다!"**
스파크는 대용량 데이터를 다루므로, 전체 개수를 세는 작업은 명시적으로 시켜야 합니다.

### ① 특정 컬럼 하나만 확인할 때 (Basic)

가장 기초적인 `filter + isNull` 패턴입니다.

>자세한 내용은 [[DataFrame_Transform#③ [핵심] 결측치(Null) 처리 (`isNotNull`,`isNull`)|결측치처리-filter]] 참조 

```python
from pyspark.sql.functions import col, isnan, when, count

# 'salary' 컬럼이 비어있는 행의 개수 세기
df.filter(col("salary").isNull()).count()
```

### ②  모든 컬럼 한눈에 확인하기 (Advanced) ️

Pandas의 `df.isna().sum()` 처럼 **모든 컬럼의 결측치**를 한 번에 보고 싶다면, **리스트 컴프리헨션(List Comprehension)** 를 써야 합니다.

```python
# "모든 컬럼을 돌면서(for c), Null이면 1을 세라(count)"
df.select([count(when(col(c).isNull(), c)).alias(c) for c in df.columns]).show()
```

---
## 결측치(Null) 처리 전략

데이터에 구멍(Null)이 뚫려있을 때, 스파크는 **`df.na`** 라는 전용 도구함(Accessor)을 제공합니다.

| 전략 | 함수 | 설명 |
| :--- | :--- | :--- |
| **지우기 (Drop)** | `df.na.drop()` | "구멍 난 데이터는 아예 안 쓸래." (데이터가 충분할 때) |
| **채우기 (Fill)** | `df.na.fill()` | "구멍을 0이나 평균값으로 메울래." (데이터가 소중할 때) |

---
## 결측치 지우기 (`drop`) 

조건을 걸어서 필요한 데이터만 남깁니다.

```python
# 1. 기본: 하나라도 Null이 있으면 그 행(Row) 삭제
df.na.drop(how="any").show()

# 2. 전부 Null일 때만 삭제 (엄격한 기준)
df.na.drop(how="all").show()

# 3. 특정 컬럼에 Null이 있을 때만 삭제
df.na.drop(subset=["salary"]).show()

# 4. [고급] 정상 데이터가 n개 미만이면 삭제 (thresh)
# "값이 2개 이상은 있어야 살려준다. (Null이 너무 많으면 탈락)"
df.na.drop(thresh=2).show()
```

### [심화] `thresh`의 진실 (생존 커트라인) 

**"몇 개가 비었냐(Null Count)"가 아니라, "몇 개가 살아있냐(Non-Null Count)"를 봅니다.** 
(Pandas와 똑같이 **행(Row) 기준**입니다!)


---
## 결측치 채우기 (`fill`) 

구멍 난 곳을 특정 값으로 덮어씁니다. (Imputation)

```python
# 1. 문자열 채우기 (String 타입 컬럼만 찾아서 채워줌)
df.na.fill("Unknown").show()

# 2. 숫자 채우기 (Integer/Double 타입만 찾아서 채워줌)
df.na.fill(0).show()

# 3. 특정 컬럼만 콕 집어서 채우기 (안전한 방법) ✅
df.na.fill("NA", subset=["occupation"]).show()
df.na.fill(0, subset=["salary"]).show()
```

###  평균값(Mean)으로 채우기 

"빈 연봉 칸을 그냥 0으로 채우면 통계가 왜곡되니까, **평균 연봉**으로 채우자!" 
-> **스텝이 2번 필요합니다.** (평균 계산 -> 채우기)

```python
from pyspark.sql import functions as F

# Step 1. 평균값 계산해서 가져오기 (collect)
# collect() 결과는 이중 리스트 형태임
mean_val = df.select(F.mean("salary")).collect()[0][0]

# Step 2. 그 값으로 채우기
df=df.na.fill(mean_val, subset=["salary"])
```

#### 🧐 분석: 왜 `collect()[0][0]` 인가요? (이중 리스트의 비밀)

`collect()`는 데이터를 **드라이버(Python)** 로 가져오는데, 이때 **`List[Row]`** 형태가 됩니다.
마치 **"상자 안의 상자"** 를 여는 것과 같습니다.

1.  **`df.select(mean).collect()`**
    * 결과: `[Row(avg=5000)]`
    * 설명: **리스트(List)** 안에 **Row 객체**가 하나 들어있음.

2.  **`.collect()[0]` (첫 번째 껍질 까기)**
    * 결과: `Row(avg=5000)`
    * 설명: 리스트에서 **첫 번째 행(Row)**을 꺼냄.

3.  **`.collect()[0][0]` (두 번째 껍질 까기)**
    * 결과: `5000`
    * 설명: 그 Row 객체에서 **첫 번째 컬럼 값(Value)** 을 꺼냄. (비로소 숫자가 됨!)

> **💡 요약:** 스파크에서 값 하나만 쏙 빼오려면 **`[0][0]` (Row 꺼내고 -> Value 꺼내고)** 패턴을 기억하세요!

#### 🧐 왜 굳이 `collect()`를 해야 하나요? (Cluster vs Driver)

`df.na.fill( df.select(mean) )` 처럼 바로 넣으면 안 되나요?
👉 **안 됩니다!** 🙅‍♂️

* **이유:** `fill()` 함수는 **"확정된 값(Literal)"** 만 받기 때문입니다.
* **과정:**
    1.  스파크(Cluster)가 평균을 계산합니다.
    2.  **`collect()`** 를 통해 그 값을 내 노트북(Driver)으로 가져와서 파이썬 변수로 만듭니다.
    3.  그 변수를 다시 `fill()` 안에 넣어서 스파크에게 명령을 내립니다.

> **💡 요약:** "평균값 하나 정도는 작으니까 드라이버로 가져와도 안전합니다."
> (전체 데이터를 collect 하는 게 위험한 것이지, 통계값 하나는 괜찮습니다!)

---
#### 🧠  중앙값(Median)과 최빈값(Mode)은 없나요?

판다스처럼 `median()`, `mode()` 함수가 바로 없습니다. 스파크 방식은 조금 다릅니다.

**1. 중앙값 (Median): `percentile_approx` 사용**

대용량 데이터에서 '정확한' 중앙값을 찾으려면 데이터를 다 정렬해야 해서 엄청 느립니다.
그래서 스파크는 **"근사치(Approximate)"** 함수를 씁니다. (오차범위 거의 없음)

```python
# 0.5 지점(50%)에 있는 값을 찾아라 = 중앙값
# 1. 계산 (collect)
median_val = df.select(
    F.percentile_approx("salary", 0.5)
).collect()[0][0]

# 2. 채우기
df = df.na.fill(median_val, subset=["salary"])

```

**최빈값 (Mode): 직접 구현해야 함** 

"가장 많이 등장한 값"을 찾는 함수가 없어서, **Group By + Count + Sort** 로 직접 찾아야 합니다.

```python
# "직업(occupation)" 컬럼의 최빈값 찾기

# 1. 그룹핑 -> 카운트 -> 내림차순 정렬 -> 맨 위 1개 가져오기(first)
# 결과: Row(occupation='Engineer', count=50)
mode_row = df.groupBy("occupation").count().orderBy(F.desc("count")).first()

# 2. 그 Row에서 값만 꺼내기
mode_val = mode_row["occupation"]

# 3. 채우기
df = df.na.fill(mode_val, subset=["occupation"])
```

---
## 날짜와 숫자 다루기 (`Date` & `Number`) 📅

날짜에서 연/월/일을 뽑아내거나, 숫자를 예쁘게 포맷팅하는 기술입니다.

### 날짜 파싱 (Date Parsing)

```python
# 'date' 컬럼이 "2023-10-25" 형태라고 가정

# 연도(Year), 월(Month), 일(Day) 추출
df.select(
    F.year("date").alias("year"),
    F.month("date").alias("month"),
    F.dayofmonth("date").alias("day") # 1~31일
).show()
```

### 숫자 포맷팅 (Number Formatting)

소수점이 너무 길 때 (`123.456789...`) 깔끔하게 자릅니다.

```python
# 소수점 2째 자리까지만 표시 (반올림 포함)
df.select(
    F.format_number("avg_score", 2).alias("score")
).show()
```

---
## [절대 원칙] Spark에는 `inplace=True`가 없다! 🚫

**"Spark DataFrame은 한 번 만들어지면 절대 수정할 수 없습니다 (Immutable)."**
Pandas를 쓰던 습관대로 코드를 짜면 아무 일도 일어나지 않습니다.

- 🔗 참고: [[Spark_DataFrame#[핵심] 불변성 (Immutability)|불변성 자세히 보기]]

### ❌ 틀린 코드 (Pandas 습관)

```python
# Pandas에서는 이게 되지만, Spark에서는 아무 효과 없음!
# (계산만 하고 결과를 허공에 날려버림)
df.na.drop()
df.withColum(...)
```

### ⭕️ 맞는 코드 (재할당 필수)

**"변경된 결과를 반드시 변수에 다시 담아야 합니다."**

```python
# 1. 같은 변수에 덮어쓰기 (가장 흔한 패턴)
df = df.na.drop()

# 2. 새로운 변수에 담기
df_clean = df.na.drop()
```

## 실전 예제: 연도별 평균 구하기 

위의 기능들을 조합하면 이런 리포트를 만들 수 있습니다.

```python
# 1. 날짜에서 '연도'를 뽑아내고 (withColumn)
# 2. 연도별로 그룹핑해서 (groupBy)
# 3. 숫자의 평균을 구하고 (mean)
# 4. 보기 좋게 소수점 자르기 (format_number)

df.withColumn("year", F.year('date')) \
  .groupBy("year") \
  .mean("number") \
  .withColumnRenamed("avg(number)", "avg") \
  .select("year", F.format_number("avg", 2).alias("avg")) \
  .show()
```

>"**평균값으로 채우기** 코드가 제일 중요해! `df.na.fill(df.mean()...)` 처럼 한 줄에 안 써진다고 당황하지 마. 
>**1. 계산해서 변수에 담고(`collect`) -
> 2. 그 변수를 넣는다(`fill`)**


---
## 🕒 날짜와 시간 타입 변환 (Date & Timestamp)

스파크에서 시간 계산이나 윈도우 집계를 하려면 반드시 **TimestampType**이어야 합니다.
하지만 CSV나 JSON에서 읽어오면 대부분 **String(문자열)** 상태이므로 변환이 필수입니다.

###  `to_timestamp()`: 문자를 시간으로 

입력된 날짜 포맷을 해석해서 Timestamp 타입으로 바꿉니다. 
포맷이 맞지 않으면 `null`을 반환하니 주의하세요.

- **문법**: `{python}to_timestamp(컬럼, "포맷")`

```python
from pyspark.sql.functions import to_timestamp, col

# 예시 데이터: "2024-05-01 12:30:00" (String)
df_cleaned = df.withColumn(
    "event_time",
    to_timestamp(col("raw_time"), "yyyy-MM-dd HH:mm:ss")
)
```

### 자주 쓰는 날짜 포맷 (Java SimpleDateFormat 기준)

스파크는 Java의 날짜 포맷을 따릅니다. 
대소문자를 틀리면 에러가 납니다!

|**기호**|**의미**|**예시**|**주의사항**|
|---|---|---|---|
|**`y`**|년 (Year)|`2024`|소문자 y|
|**`M`**|월 (Month)|`05`|**대문자 M** (소문자 m은 '분'!)|
|**`d`**|일 (Day)|`01`|소문자 d|
|**`H`**|시 (Hour 0-23)|`14`|**대문자 H** (소문자 h는 1-12)|
|**`m`**|분 (Minute)|`30`|소문자 m|
|**`s`**|초 (Second)|`59`|소문자 s|

**꿀팁:** 포맷을 생략하면 기본적으로 `"yyyy-MM-dd HH:mm:ss"` 형식을 기대하고 변환합니다.

---
### 💡 왜 `to_date`가 아니라 `to_timestamp`인가요?

-  **`to_date`**: 연-월-일(`2024-01-01`)까지만 남기고 시간은 버립니다.
- **`to_timestamp`**: 연-월-일 시:분:초(`2024-01-01 12:00:00`)까지 모두 살립니다.
- **스트리밍 집계에서는 초 단위가 중요하므로 `to_timestamp`를 써야 합니다.**