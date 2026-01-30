---
aliases:
  - Pandas Null
  - NaN
  - 결측치
  - fillna
  - dropna
  - isna
tags:
  - Python
  - Pandas
related:
  - "[[Python_Lists_Tuples]]"
  - "[[Spark_Data_Cleaning]]"
  - "[[00_Pandas_HomePage]]"
linked:
  - file:////Users/gong/gong_study_de/pandas/notebooks/missing_value.ipynb
---
##  개념 한 줄 요약 

**"비어있는 값(`NaN`, `None`)을 찾아서, 지워버리거나(`drop`) 다른 값으로 메우는(`fill`) 과정."**

* **NaN (Not a Number):** 판다스에서 결측치를 표현하는 특수한 값 (실수형 Float 취급).
* **전략:** 데이터가 충분하면 **삭제**, 데이터가 적으면 **평균/최빈값 대치**.

---
## 결측치 확인하기 (`isna`) 

데이터를 받자마자 가장 먼저 해야 할 일입니다.

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'name': ['IronMan', 'Hulk', np.nan, 'Thor'],
    'age': [30, 45, np.nan, np.nan],
    'score': [100, 90, 80, np.nan]
})

# 1. True/False 마스크로 확인
print(df.isna()) 

# 2. [실무 필수] 컬럼별 결측치 개수 세기 (sum)
print(df.isna().sum())
# name     1
# age      2
# score    1
```

**💡 참고:** `isnull()`은 `isna()`와 완전히 똑같은 함수입니다.

---
## 결측치 지우기 (`dropna`)

쿨하게 버리는 방법입니다.

```python
# 1. 하나라도 NaN이 있으면 행(Row) 삭제 (기본값)
df.dropna(how='any')

# 2. 모든 값이 NaN일 때만 행 삭제
df.dropna(how='all')

# 3. 특정 컬럼에 NaN이 있을 때만 삭제 (subset)
df.dropna(subset=['name', 'score'])

# 4. [고급] 정상 데이터가 n개 이상이어야 살림 (thresh)
# "값이 적어도 2개는 채워져 있어야 데이터를 인정해줄게."
# (값이 1개뿐이고 나머지가 다 NaN이면 삭제됨)
df.dropna(thresh=2)

# 5. [중요] 원본에 바로 반영하기 (inplace)
df.dropna(subset=['name', 'score'], how='all', inplace=True)
```

**⚠️ 주의:** `inplace=True`를 안 쓰면 삭제된 결과만 보여주고 원본은 그대로입니다!

💡 Thresh 헷갈리지 말기:
- `thresh=2`는 "NaN이 2개면 지워라"가 **아닙니다!**
- **"정상적인 값(Non-NaN)이 최소 2개는 살아있어야 한다"** 는 뜻입니다. (생존 커트라인)

---
### `subset` 사용할 때 주의할 점 (`any` vs `all`)

"왜 하나만 NaN이어도 지워지나요? 억울해요!" 

```python
# 상황: subset을 썼는데 데이터가 다 날아갔어요.
df.dropna(subset=['name', 'score'])
```

- **이유:** `dropna`의 기본 설정이 **`how='any'`** 이기 때문입니다.
-  **해석:** "`name`과 `score` 중 **하나라도(any)** 없으면 탈락시킨다." (엄격한 기준)
- 해결책: 둘 다 없을 때만 지우고 싶다면? **`how='all'`** 을 명시해줘야 합니다.

```python
# 해석: "name과 score가 '모두(all)' 없을 때만 지워라!"
# (하나라도 살아있으면 살려줌)
df.dropna(subset=['name', 'score'], how='all')
```

---
## 4. 결측치 채우기 (`fillna`) 

삭제하기엔 데이터가 너무 소중할 때, 가장 합리적인 값으로 구멍을 메웁니다.

### ① 특정 값으로 채우기 

가장 단순한 방법입니다.

```python
# 1. 0으로 채우기 (숫자 데이터)
df.fillna(0)

# 2. "Unknown"으로 채우기 (문자 데이터)
df.fillna("Unknown")

# 3. [실무] 컬럼별로 다르게 채우기 (Dictionary)
df.fillna({
    'name': 'Anonymous',  # 이름은 '익명'으로
    'age': 0,             # 나이는 0으로
    'score': 50           # 점수는 기본점수로
})
```

### ② 통계값으로 채우기 (Statistical Imputation)

**"그냥 0으로 채우면 데이터가 왜곡됩니다."** 
데이터의 특성에 따라 평균, 중앙값, 최빈값을 골라 써야 합니다.

| 방법               | 함수                      | 적용 대상 (실무 룰)         | 이유 (Why)                                               |
| :--------------- | :---------------------- | :------------------- | :----------------------------------------------------- |
| **평균 (Mean)**    | `.mean()`               | **키, 몸무게, 시험 점수**    | 데이터가 종 모양(정규분포)이라서 평균이 가장 무난한 대푯값임.                    |
| **중앙값 (Median)** | `.median()`             | **연봉, 집값, 자산, 매출**   | **돈(Money)** 관련 데이터는 슈퍼 부자(Outlier)가 평균을 왜곡하기 때문!      |
| **최빈값 (Mode)**   | `.mode()[0]`            | **혈액형, 지역, 등급**      | 더하고 나눌 수 없는 **문자열(Category)** 데이터니까.                   |
| **앞/뒤 채우기**      | `.ffill()` / `.bfill()` | **주식, 환율, 상태값**      | **"어제 가격"**이 오늘 가격과 가장 비슷하니까. (지속성)                    |
| **보간법**          | `.interpolate()`        | **센서, GPS, 온도(시계열)** | **"자연은 점프하지 않는다."** 1시가 10도, 3시가 30도면 2시는 20도일 확률이 높음. |

### [심화] 센서 데이터: 평균 vs 보간법?

" 센서가 평균이라면서요?" -> **목적에 따라 다릅니다!**

1. **단순 요약 (평균):** "오늘 하루 평균 온도가 몇 도야?" → `mean()`
2. **그래프 복원 (보간법):** "오후 2시에 잠깐 센서가 꺼졌는데, 그때 몇 도였을까?" → `interpolate()`

> **💡 실무 팁:** 시계열 데이터(시간 흐름이 있는 데이터)라면 **무조건 보간법이나 앞/뒤 채우기**가 평균보다 정확합니다!


```python
# 1. 평균(Mean) 대치
df['age'] = df['age'].fillna(df['age'].mean())

# 2. 중앙값(Median) 대치 (연봉 100억인 사람이 평균을 망칠 때 사용)
df['salary'] = df['salary'].fillna(df['salary'].median())

# 3. 최빈값(Mode) 대치 (가장 많이 등장한 혈액형 등으로 채움)
# 주의: mode()는 시리즈를 반환하므로 [0]으로 첫 번째 값을 꺼내야 함!
most_freq = df['blood_type'].mode()[0]
df['blood_type'] = df['blood_type'].fillna(most_freq)
```

### ③ 앞/뒤 값으로 채우기 (Time Series)

시계열 데이터(주식, 날씨)에서 **"어제 가격"** 이나 **"내일 날씨"** 로 빈칸을 채울 때 씁니다.
(Pandas 2.0부터는 `fillna(method=...)`가 삭제되었으므로, **전용 함수**를 써야 합니다.)

```python
# ffill (Forward Fill): 앞의 값으로 뒤를 채움 (어제 데이터를 오늘로 복사)
# (구버전: df.fillna(method='ffill'))
df.ffill()

# bfill (Backward Fill): 뒤의 값으로 앞을 채움
# (구버전: df.fillna(method='bfill'))
df.bfill()

# [꿀팁] 무한정 채우지 않기 (limit)
# 결측치가 100개 연속으로 있어도 딱 1개만 채우고 멈춤
df.ffill(limit=1)
```

### ④ 선형 보간 (`interpolate`) 

단순 복사가 아니라, **"점과 점 사이를 선으로 이어서"** 자연스럽게 채웁니다.
(`fillna`와 달리, 여기서는 **`method` 파라미터가 필수**입니다!)

```python
# 1. 선형 보간 (Linear): 기본값
# (10, NaN, 30) -> 20
df.interpolate(method='linear')

# 2. 그 외 (Polynomial, Spline 등)
# 곡선 형태로 부드럽게 채울 때 사용 (order=몇 차 함수인지 지정 필요)
df.interpolate(method='polynomial', order=2)
```

#### ⭐️ [중요] 시계열 보간 (`method='time'`) 의 필수 조건

날짜 간격이 불규칙할 때(예: 1일, 10일, 11일), **시간의 거리**를 계산해서 정확하게 채웁니다. 
단, **반드시 날짜 컬럼이 인덱스(Index)여야 합니다!** (안 그러면 `ValueError` 발생 🚨)

```python
# [에러 해결법]
# 1. 날짜 컬럼을 진짜 날짜(datetime)로 변환
df['date'] = pd.to_datetime(df['date'])

# 2. 날짜를 인덱스로 설정 (필수!) ⭐️
df = df.set_index('date')

# 3. 이제 'time' 보간이 작동함 (시간 간격에 맞춰서 채워짐)
df = df.interpolate(method='time')

# (선택) 다시 인덱스를 컬럼으로 되돌리기
df = df.reset_index()
```




---
## Spark vs Pandas 비교

|**기능**|**Pandas (import pandas)**|**Spark (import pyspark)**|
|---|---|---|
|**확인**|`df.isna().sum()`|`df.filter(col("x").isNull()).count()`|
|**삭제**|`df.dropna(subset=['col'])`|`df.na.drop(subset=['col'])`|
|**채우기**|`df.fillna({'col': 0})`|`df.na.fill(0, subset=['col'])`|
|**특징**|`inplace=True` 가능|**불변(Immutable)**. 항상 변수에 재할당해야 함|


---

>"이제 완벽해! 특히 **'최빈값(Mode)'** 쓸 때 `mode()[0]`이라고 인덱싱해야 한다는 거, 이거 실무에서 에러 정말 많이 내는 부분이니까 꼭 기억해둬. 


