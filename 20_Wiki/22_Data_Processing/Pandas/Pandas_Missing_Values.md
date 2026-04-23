---
aliases:
  - Pandas Null
  - NaN
  - 결측치
  - fillna
  - dropna
  - isna
  - unique
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Spark_Data_Cleaning]]"
linked:
  - file:////Users/gong/gong_study_de/pandas/notebooks/missing_value.ipynb
---
# Pandas_Missing_Values

> **"비어있는 값(NaN, None)을 찾아서, 지워버리거나(drop) 다른 값으로 메우는(fill) 과정."**

```
NaN (Not a Number): 판다스에서 결측치를 표현하는 특수한 값 (Float 취급)
전략: 데이터가 충분하면 삭제, 데이터가 적으면 평균/최빈값 대치
```

---

## ① 결측치 확인: `isna()`

데이터 받자마자 가장 먼저 해야 할 일이에요.

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'name':  ['IronMan', 'Hulk', np.nan, 'Thor'],
    'age':   [30, 45, np.nan, np.nan],
    'score': [100, 90, 80, np.nan]
})

# True/False 마스크로 확인
print(df.isna())

# 실무 필수: 컬럼별 결측치 개수
print(df.isna().sum())
# name     1
# age      2
# score    1
```

> `isnull()` 은 `isna()` 와 완전히 동일한 함수예요.

---

## ② 결측치 삭제: `dropna()`

```python
# 하나라도 NaN이 있으면 행 삭제 (기본값: how='any')
df.dropna()

# 모든 값이 NaN일 때만 삭제
df.dropna(how='all')

# 특정 컬럼에 NaN이 있을 때만 삭제
df.dropna(subset=['name', 'score'])

# 정상 값이 n개 이상이어야 살림 (thresh)
df.dropna(thresh=2)
# "Non-NaN 값이 최소 2개는 있어야 한다" (NaN 2개가 아님!)

# 원본에 바로 반영
df.dropna(subset=['name'], inplace=True)
```

> ⚠️ `inplace=True` 없으면 원본은 그대로예요. 결과만 보여주고 끝.

### subset + how 조합

```python
# how='any' (기본): name 또는 score 중 하나라도 NaN이면 삭제 (엄격)
df.dropna(subset=['name', 'score'], how='any')

# how='all': name과 score 둘 다 NaN일 때만 삭제 (관대)
df.dropna(subset=['name', 'score'], how='all')
```

---

## ③ 결측치 채우기: `fillna()`

### 특정 값으로 채우기

```python
df.fillna(0)              # 숫자 데이터
df.fillna("Unknown")      # 문자 데이터

# 컬럼별로 다르게 채우기
df.fillna({
    'name':  'Anonymous',
    'age':   0,
    'score': 50
})
```

### 통계값으로 채우기

|방법|함수|적용 대상|이유|
|---|---|---|---|
|평균 (Mean)|`.mean()`|키, 몸무게, 점수|정규분포 데이터의 무난한 대푯값|
|중앙값 (Median)|`.median()`|연봉, 집값, 자산|이상치(부자)가 평균을 왜곡하기 때문|
|최빈값 (Mode)|`.mode()[0]`|혈액형, 지역, 등급|더하고 나눌 수 없는 문자열 데이터|
|앞/뒤 채우기|`.ffill()` / `.bfill()`|주식, 환율, 상태값|어제 가격이 오늘과 가장 비슷|
|보간법|`.interpolate()`|센서, GPS, 온도|자연은 점프하지 않는다|

```python
# 평균 대치
df['age'] = df['age'].fillna(df['age'].mean())

# 중앙값 대치 (연봉처럼 이상치 있을 때)
df['salary'] = df['salary'].fillna(df['salary'].median())

# 최빈값 대치 (문자 컬럼)
# ⚠️ mode()는 Series 반환 → [0]으로 첫 번째 값 꺼내야 함
most_freq = df['blood_type'].mode()[0]
df['blood_type'] = df['blood_type'].fillna(most_freq)
```

### 앞/뒤 값으로 채우기 (시계열)

```python
# ffill (Forward Fill): 앞의 값으로 뒤를 채움
df.ffill()

# bfill (Backward Fill): 뒤의 값으로 앞을 채움
df.bfill()

# 최대 n개만 채우고 멈춤
df.ffill(limit=1)
```

> ⚠️ Pandas 2.0부터 `fillna(method='ffill')` 삭제됨 → `df.ffill()` / `df.bfill()` 전용 함수 사용

### 보간법: `interpolate()`

단순 복사가 아니라 **점과 점 사이를 선으로 이어서** 자연스럽게 채워요.

```python
# 선형 보간 (10, NaN, 30) → 20
df.interpolate(method='linear')

# 시계열 보간 (날짜 간격 불규칙할 때)
# ⚠️ 반드시 날짜 컬럼이 인덱스여야 함!
df['date'] = pd.to_datetime(df['date'])
df = df.set_index('date')
df = df.interpolate(method='time')
df = df.reset_index()  # 필요하면 다시 컬럼으로
```

---

## 센서 데이터: 평균 vs 보간법?

```
단순 요약 → mean()
"오늘 하루 평균 온도가 몇 도야?"

시계열 복원 → interpolate()
"오후 2시에 센서가 꺼졌는데, 그때 몇 도였을까?"

→ 시간 흐름이 있는 데이터면 보간법이 평균보다 정확
```

---

## 이 과제에서 쓴 패턴

```python
# NCS 수행준거에서 빈값 제거 후 번역
ncs_kos = df_business["수행준거"].dropna().unique()

# 빈값 제거 후 상위 10개 확인
df_business["지식기술태도의의"].dropna().head(10).tolist()
```

---

## Pandas vs Spark 비교

|기능|Pandas|Spark|
|---|---|---|
|확인|`df.isna().sum()`|`df.filter(col("x").isNull()).count()`|
|삭제|`df.dropna(subset=['col'])`|`df.na.drop(subset=['col'])`|
|채우기|`df.fillna({'col': 0})`|`df.na.fill(0, subset=['col'])`|
|특징|`inplace=True` 가능|불변(Immutable) → 항상 변수에 재할당|

---
