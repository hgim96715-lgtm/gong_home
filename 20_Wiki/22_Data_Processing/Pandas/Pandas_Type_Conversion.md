---
aliases:
  - astype
  - to_datetiem
  - to_numeric
  - 타입변환
  - 형 변환
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Pandas_DataStructures]]"
  - "[[Pandas_Missing_Values]]"
---

# Pandas_Type_Conversion — 타입 변환

## 한 줄 요약

```
astype()      → 컬럼 타입을 원하는 타입으로 변환
to_datetime() → 문자열 → 날짜 타입
to_numeric()  → 문자열 → 숫자 타입
errors='coerce' → 변환 실패 시 NaN (에러 없이 처리)
```

---

---

# ① 타입 확인 먼저

```python
df.dtypes           # 전체 컬럼 타입 확인
df["col"].dtype     # 특정 컬럼 타입
df.info()           # 타입 + 결측치 + 메모리 한 번에
```

```
자주 보는 dtype:
  object   = 문자열 (str)
  int64    = 정수
  float64  = 실수
  bool     = 불리언
  datetime64[ns] = 날짜
  category = 카테고리 (메모리 절약)
```

---

---

# ② astype() — 타입 변환 ⭐️

```python
df["col"].astype(타입)
```

## 기본 사용

```python
df["age"] = df["age"].astype(int)       # 정수로
df["price"] = df["price"].astype(float) # 실수로
df["name"] = df["name"].astype(str)     # 문자열로
df["flag"] = df["flag"].astype(bool)    # 불리언으로

# 카테고리로 (메모리 절약 — 반복 값 많을 때)
df["gender"] = df["gender"].astype("category")
```

## 여러 컬럼 한 번에

```python
df = df.astype({
    "age": int,
    "price": float,
    "name": str
})
```

## 실전 패턴 — 텍스트 컬럼 합치기 ⭐️

```python
# 문제: Title / Description 이 NaN 섞여있고 타입이 object
# NaN 이 있으면 문자열 합칠 때 "nan" 이 섞여 들어감

# 1단계: NaN 행 제거
onet_texts = occupation_df.dropna(subset=["Title", "Description"]).copy()

# 2단계: astype(str) 로 안전하게 문자열 변환 후 합치기
onet_texts["text"] = (
    onet_texts["Title"].astype(str)
    + " "
    + onet_texts["Description"].astype(str)
)
```

```
.copy() 를 붙이는 이유:
  dropna() 결과는 원본의 뷰(View) 일 수 있음
  .copy() 로 독립 복사본 생성 → SettingWithCopyWarning 방지

astype(str) 이 필요한 이유:
  object 타입 컬럼이라도 실제로 숫자/None 이 섞일 수 있음
  astype(str) → 전부 문자열로 통일 후 + 연산
  NaN 이 남아있으면 "nan" 문자열이 됨 → 그래서 dropna 먼저
```

## 주의 — astype 실패 시 에러

```python
# 변환 불가한 값이 있으면 에러
df["age"] = df["age"].astype(int)
# ValueError: invalid literal for int() with base 10: 'N/A'

# 해결: errors='ignore' (실패하면 원본 유지)
df["age"] = df["age"].astype(int, errors='ignore')

# 또는 to_numeric 으로 errors='coerce' 사용 (아래 참고)
```

---

---

# ③ pd.to_numeric() — 숫자 변환 ⭐️

```python
pd.to_numeric(시리즈, errors='raise'|'coerce'|'ignore')
```

```python
import pandas as pd

s = pd.Series(["1", "2", "abc", "4.5", None])

# 기본 (에러 발생)
pd.to_numeric(s)
# ValueError: unable to parse string "abc"

# errors='coerce' — 변환 실패 → NaN (가장 많이 씀)
pd.to_numeric(s, errors='coerce')
# 0    1.0
# 1    2.0
# 2    NaN   ← "abc" 변환 실패 → NaN
# 3    4.5
# 4    NaN   ← None → NaN

# errors='ignore' — 변환 실패 → 원본 유지
pd.to_numeric(s, errors='ignore')
# 0      1
# 1      2
# 2    abc   ← 그대로 유지
# 3    4.5
# 4   None
```

## errors 옵션 비교

```
errors='raise'  (기본값):
  변환 실패 → ValueError 발생
  데이터가 확실히 깨끗할 때

errors='coerce':
  변환 실패 → NaN 으로 대체
  지저분한 실전 데이터 처리에 가장 많이 씀

errors='ignore':
  변환 실패 → 원본 값 유지
  일부만 변환해도 괜찮을 때
```

## downcast — 메모리 최적화

```python
# 정수로 변환하면서 메모리 최소화
pd.to_numeric(s, errors='coerce', downcast='integer')
# int64 → int8/int16/int32 로 자동 축소

pd.to_numeric(s, errors='coerce', downcast='float')
# float64 → float32 로 축소
```

---

---

# ④ pd.to_datetime() — 날짜 변환 ⭐️

```python
pd.to_datetime(시리즈, format=None, errors='coerce')
```

## 기본 사용

```python
df["date"] = pd.to_datetime(df["date"])
# "2024-01-15" → Timestamp('2024-01-15 00:00:00')

# 포맷 지정 (더 빠름)
df["date"] = pd.to_datetime(df["date"], format="%Y-%m-%d")
df["date"] = pd.to_datetime(df["date"], format="%Y%m%d")      # 20240115
df["date"] = pd.to_datetime(df["date"], format="%d/%m/%Y")    # 15/01/2024
```

## errors='coerce' — 변환 실패 NaT

```python
s = pd.Series(["2024-01-15", "invalid", "2024-03-20"])

pd.to_datetime(s, errors='coerce')
# 0   2024-01-15
# 1          NaT   ← 변환 실패 → NaT (Not a Time)
# 2   2024-03-20
```

```
NaN vs NaT:
  NaN  = 숫자 / 문자열의 결측값
  NaT  = 날짜 타입의 결측값 (Not a Time)
  둘 다 isnull() 로 감지 가능
```

## 날짜 변환 후 활용

```python
df["date"] = pd.to_datetime(df["date"])

# 날짜 구성요소 추출
df["year"]  = df["date"].dt.year
df["month"] = df["date"].dt.month
df["day"]   = df["date"].dt.day
df["weekday"] = df["date"].dt.day_name()  # Monday, Tuesday ...

# 날짜 차이 계산
df["days_since"] = (pd.Timestamp.now() - df["date"]).dt.days

# 날짜 필터링
df[df["date"] >= "2024-01-01"]
```

---

---

# ⑤ 전체 비교 정리

|상황|방법|
|---|---|
|타입 강제 변환|`astype(int/float/str)`|
|여러 컬럼 한번에|`astype({"col1": int, "col2": str})`|
|숫자 변환 + 실패 처리|`pd.to_numeric(col, errors='coerce')`|
|날짜 변환 + 실패 처리|`pd.to_datetime(col, errors='coerce')`|
|텍스트 합치기 전 안전 변환|`col.astype(str)`|
|메모리 절약|`astype("category")` 또는 `downcast`|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`astype(int)` 에러|NaN 또는 변환 불가값|`pd.to_numeric(errors='coerce')` 후 fillna|
|문자열 합치기에서 "nan" 이 섞임|NaN 남아있는 상태로 합침|`dropna()` 먼저 / `fillna("")` 후 합치기|
|`to_datetime` 후 타입 여전히 object|format 불일치|`format=` 명시 또는 `errors='coerce'`|
|`.copy()` 없이 dropna 후 수정|SettingWithCopyWarning|`dropna(...).copy()`|
|downcast 후 overflow|값 범위 초과|downcast 전 값 범위 확인|