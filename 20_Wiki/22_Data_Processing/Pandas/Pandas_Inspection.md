---
aliases:
  - head
  - info
  - describe
  - shape
  - tail
  - 데이터확인
  - 데이터요약
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Spark_DataFrame]]"
  - "[[Pandas_Groupby]]"
  - "[[Pandas_Missing_Values]]"
---
# Pandas_Inspection

> **"데이터를 요리하기 전에, 재료가 상했는지(Null) 양은 얼마나 되는지(Shape) 확인하는 필수 절차."**

---

## 필수 검사 순서

```
shape (몇 개지?) → head (어떻게 생겼지?) → info (타입은 맞나?) → describe (이상한 값은 없나?)
```

---

## ① 크기 확인: `.shape`

가장 먼저 행(Row)과 열(Column)의 개수를 봐요.

```python
import pandas as pd

df = pd.read_csv("data.csv")
print(df.shape)
# (1000, 5) → 행 1000개, 열 5개
```

> ⚠️ `shape`는 **속성(Attribute)** 이라 `()` 를 안 붙여요. `df.shape()` → 에러 / `df.shape` → 정상

---

## ② 눈으로 보기: `.head()` / `.tail()` / `.sample()`

```python
df.head()           # 맨 위 5줄 (기본값)
df.head(3)          # 맨 위 3줄

df.tail()           # 맨 아래 5줄
df.tail(3)          # 맨 아래 3줄

df.sample(3)        # 무작위 3개 (편향 없이 확인)
df.sample(frac=0.5) # 전체의 50% 무작위 추출
df.sample(3, random_state=42)  # 시드 고정: 언제 실행해도 같은 결과
```

|메서드|언제 쓰나|
|---|---|
|`head()`|헤더가 잘 들어갔는지 확인|
|`tail()`|데이터가 끝까지 잘 들어왔는지 확인|
|`sample()`|데이터가 골고루 섞여 있는지 편향 없이 확인|

### Spark와 차이점

```
Pandas: sample(n=3)    → 개수 기준 (기본)
Spark:  sample(fraction=0.1) → 비율 기준 (기본)
        개수로 뽑으려면 takeSample() 사용
```

---

## ③ 뼈대 확인: `.info()` ⭐ 가장 중요

데이터의 **주민등록초본** 같은 것이에요.

```python
df.info()
# <class 'pandas.core.frame.DataFrame'>
# RangeIndex: 1000 entries, 0 to 999
# Data columns (total 5 columns):
#  #   Column   Non-Null Count  Dtype
# ---  ------   --------------  -----
#  0   Name     1000 non-null   object
#  1   Age       990 non-null   float64  ← 10개 비었음! (결측치)
#  2   Salary   1000 non-null   int64
```

### 확인할 것 4가지

```
1. Non-Null Count : 결측치 있나? (전체보다 적으면 구멍 난 것)
2. Dtype          : 숫자가 문자(object)로 잘못 잡히지 않았나?
3. Memory Usage   : 메모리를 얼마나 잡아먹나?
4. Column         : 컬럼 이름 오타 없나?
```

---

## ④ 통계 요약: `.describe()`

수치형 데이터 분포를 한눈에 봐요. "나이가 200살? 월급이 마이너스?" 같은 **이상치(Outlier)** 찾을 때 써요.

```python
# 숫자 컬럼만 요약 (기본)
print(df.describe())

# 문자 컬럼 포함 전체 요약
print(df.describe(include='all'))
# 문자 컬럼: unique(고유값 수), top(가장 흔한 값), freq(빈도) 추가로 나옴
```

### Spark와 차이점

```python
# Pandas
df[["Age"]].describe()     # 컬럼 먼저 선택 후 describe

# Spark
df.describe("Age").show()  # 괄호 안에 컬럼명 넣음

# ⚠️ Pandas에서 describe("Age") 하면 에러!
```

---

## ⑤ 고유값 확인: `.unique()` / `.nunique()`

```python
# unique(): 어떤 값들이 있는지 목록 확인
df["지식기술태도코드명"].unique()
# ['지식', '기술', '태도']

# nunique(): 고유값이 몇 개인지 숫자로 확인
df["지식기술태도코드명"].nunique()
# 3
```

> `unique()`는 새 데이터 처음 볼 때 **반드시** 쓰는 메서드예요. 어떤 카테고리값이 있는지 모르면 분석을 시작할 수 없어요.

---

## ⑥ 정렬 후 상위 N개: `.sort_values()` + `.head()`

```python
# 유사도 높은 순으로 상위 10개
df.sort_values("유사도", ascending=False).head(10)
#              ↑ 정렬 기준 컬럼    ↑ 내림차순

# ascending=True  → 오름차순 (작은 것부터)
# ascending=False → 내림차순 (큰 것부터, 기본 사용 패턴)
```

---

## Pandas vs Spark 비교

|기능|Pandas|Spark|
|---|---|---|
|행/열 개수|`df.shape` (속성, 즉시)|`(df.count(), len(df.columns))` (느림)|
|눈으로 보기|`df.head()` (HTML 표)|`df.show()` (텍스트)|
|구조 확인|`df.info()`|`df.printSchema()`|
|통계 확인|`df.describe()`|`df.describe().show()`|

---
