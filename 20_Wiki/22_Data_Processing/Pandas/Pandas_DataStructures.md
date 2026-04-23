---
aliases:
  - Series
  - DataFrame
  - Schema
  - 판다스구조
  - 데이터프레임
  - 시리즈
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Pandas_Type_Conversion]]"
  - "[[Pandas_Missing_Values]]"
---

---

# Pandas_DataStructures — Series / DataFrame 구조

## 한 줄 요약

```
Series    = 엑셀의 열(Column) 하나  (1차원)
DataFrame = 엑셀의 시트 전체        (2차원)
DataFrame 에서 열 하나 뽑으면 → Series
```

---

---

# ① Series — 1차원 데이터

```
Index + Value 가 한 쌍
리스트와 비슷하지만 인덱스(이름표)가 붙어있음
```

```python
import pandas as pd

# 기본 생성 (인덱스 자동 0,1,2...)
data = [100, 200, 300]
s = pd.Series(data)
# 0    100
# 1    200
# 2    300
# dtype: int64

# 인덱스 직접 지정
s = pd.Series(data, index=["A", "B", "C"])
# A    100
# B    200
# C    300

# 딕셔너리로 생성
s = pd.Series({"name": "Ironman", "age": 45})
# name    Ironman
# age          45
```

## Series 조회

```python
s["A"]      # 100  (이름으로 조회)
s[0]        # 100  (위치로 조회)
s[["A", "C"]]  # A, C 만 추출
```

---

---

# ② DataFrame — 2차원 데이터 ⭐️

```
여러 개의 Series 가 옆으로 붙어서 만들어진 것
Index (행 이름) + Columns (열 이름) + Values (데이터)
```

## 생성 방법

```python
# 딕셔너리로 생성 (가장 많이 씀)
data = {
    "Name":   ["Iron Man", "Captain", "Thor"],
    "Age":    [45, 100, 1500],
    "Weapon": ["Suit", "Shield", "Hammer"]
}
df = pd.DataFrame(data)
#        Name   Age  Weapon
# 0  Iron Man    45    Suit
# 1   Captain   100  Shield
# 2      Thor  1500  Hammer

# 리스트 of 딕셔너리
rows = [
    {"Name": "Iron Man", "Age": 45},
    {"Name": "Captain",  "Age": 100},
]
df = pd.DataFrame(rows)

# CSV 읽기
df = pd.read_csv("file.csv")
```

---

---

# ③ 컬럼 선택 ⭐️

## 단일 컬럼 → Series 반환

```python
df["Name"]
# 0    Iron Man
# 1     Captain
# 2        Thor
# Name: Name, dtype: object
# ↑ Series 반환 (1차원)

type(df["Name"])   # <class 'pandas.core.series.Series'>
```

## 이중 대괄호 → DataFrame 반환 ⭐️

```python
# [[ ]] 이중 대괄호 = 여러 컬럼 선택 → DataFrame 유지
df[["Name", "Age"]]
#        Name   Age
# 0  Iron Man    45
# 1   Captain   100
# 2      Thor  1500
# ↑ DataFrame 반환 (2차원, 표 형태)

type(df[["Name"]])   # <class 'pandas.core.frame.DataFrame'>
type(df["Name"])     # <class 'pandas.core.series.Series'>
```

```
단일 대괄호 vs 이중 대괄호:
  df["Name"]         → Series  (1차원, 표가 아님)
  df[["Name"]]       → DataFrame (2차원, 표 형태)
  df[["A", "B"]]     → 두 컬럼만 가진 DataFrame

실전 활용:
  onet_texts[["Title", "Description"]].head()
  → Title, Description 두 컬럼만 표로 보기
  → head() 로 상위 5행 확인
```

## 실전 패턴

```python
# 특정 컬럼만 보기
onet_texts[["Title", "Description"]].head()
#     Title                    Description
# 0   Software Developer       Develops software...
# 1   Data Analyst             Analyzes data...

# 여러 컬럼으로 새 DataFrame 만들기
subset = df[["Name", "Weapon"]]

# 특정 컬럼 제외
df.drop(columns=["Age"])              # Age 제거
df.drop(columns=["Age", "Weapon"])    # 여러 개 제거
```

---

---

# ④ 행 선택 — iloc / loc ⭐️

```
iloc = 위치(숫자)로 선택
loc  = 이름(레이블)으로 선택
```

```python
df.iloc[0]         # 0번째 행 (Series 반환)
df.iloc[0:2]       # 0~1번째 행 (DataFrame 반환)
df.iloc[[0, 2]]    # 0번, 2번 행

df.loc[0]          # 인덱스 이름이 0 인 행
df.loc[0:2]        # 인덱스 0~2 (끝 포함! iloc 과 다름)

# 행 + 열 동시 선택
df.iloc[0, 1]          # 0번행 1번열 값 하나
df.iloc[:, 0]          # 모든 행의 0번열
df.iloc[0:2, 0:2]      # 0~1행, 0~1열
df.loc[:, ["Name", "Age"]]  # 모든 행의 Name, Age 열
```

```
iloc vs loc 끝 인덱스 차이:
  df.iloc[0:2] → 0, 1  (2 미포함 — 파이썬 슬라이싱과 동일)
  df.loc[0:2]  → 0, 1, 2  (2 포함 — 끝 인덱스 포함!)
```

---

---

# ⑤ 기본 정보 확인

```python
df.shape        # (행 수, 열 수)  ex: (3, 3)
df.columns      # 컬럼 이름 목록
df.index        # 행 인덱스 목록
df.dtypes       # 각 컬럼의 데이터 타입
df.info()       # 타입 + Non-Null + 메모리 한 번에

df.head()       # 상위 5행 (기본값)
df.head(10)     # 상위 10행
df.tail()       # 하위 5행
df.sample(5)    # 랜덤 5행
```

## info() 출력 해석

```python
df.info()
# <class 'pandas.core.frame.DataFrame'>
# RangeIndex: 3 entries, 0 to 2
# Data columns (total 3 columns):
#  #   Column  Non-Null Count  Dtype
# ---  ------  --------------  -----
#  0   Name    3 non-null      object   ← 문자열
#  1   Age     3 non-null      int64    ← 정수
#  2   Weapon  3 non-null      object

# Non-Null Count < 전체 행 수 → 결측치 있음
```

---

---

# ⑥ Series vs DataFrame 비교 ⭐️

|구분|Series|DataFrame|
|---|---|---|
|차원|1차원|2차원|
|구성|Index + Value|Index + Columns + Value|
|타입|한 종류만|컬럼마다 다를 수 있음|
|비유|엑셀 A열 하나|엑셀 시트 전체|
|선택 방법|`s["A"]`|`df["col"]` / `df[["col1","col2"]]`|

```python
# DataFrame 에서 열 하나 뽑으면 Series
type(df["Name"])       # Series
type(df[["Name"]])     # DataFrame

# 확인하는 습관
print(type(변수명))
print(변수명.shape)    # (행수,) = Series / (행수, 열수) = DataFrame
```

---

---

# ⑦ Pandas vs Spark 비교

|특징|Pandas 🐼|Spark ⚡️|
|---|---|---|
|위치|내 컴퓨터 RAM|여러 서버에 분산|
|인덱스|자동 부여 (0,1,2...)|기본적으로 없음|
|수정|가능 (Mutable)|불가 (Immutable)|
|실행|즉시 실행|지연 실행 (Lazy)|
|데이터 크기|수 GB 이내|TB, PB 단위|
|행 접근|`iloc[0]`|filter 로 조건 검색|

```
Spark 쓸 때 마인드:
  인덱스 없이 SQL 처럼 조건(filter) 으로 찾는다
  "몇 번째 줄" 개념 대신 "어떤 조건에 맞는 행" 으로 사고
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`df["A", "B"]` 에러|이중 대괄호 없음|`df[["A", "B"]]`|
|Series 인데 DataFrame 함수 쓰면 에러|타입 혼동|`type(변수)` 확인|
|`df.iloc[0:2]` 와 `df.loc[0:2]` 결과 다름|끝 인덱스 포함 여부|iloc=미포함 / loc=포함|
|`.head()` 후 원본 바뀔까봐 걱정|head 는 조회용|원본 변경 없음|