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
  - "[[Spark_DataFrame_Basics]]"
  - "[[Pandas_Read_Write]]"
  - "[[Pandas_DataStructures]]"
  - "[[00_Pandas_HomePage]]"
---
## 개념 한 줄 요약 

**"데이터를 요리하기 전에, 재료가 상했는지(Null) 양은 얼마나 되는지(Shape) 확인하는 필수 절차."**

* **순서:** `shape` (몇 개지?) -> `head` (어떻게 생겼지?) -> `info` (타입은 맞나?) -> `describe` (이상한 값은 없나?)

---
##  필수 검사 4단계 

### ① 크기 확인: `.shape` (괄호 없음!)

가장 먼저 행(Row)과 열(Column)의 개수를 봅니다.
* **주의:** 함수가 아니라 **속성(Attribute)** 이라서 `()`를 안 씁니다.

```python
import pandas as pd
df = pd.read_csv("data.csv")

print(df.shape)
# 결과: (1000, 5) -> 행 1000개, 열 5개
```

### ② 눈으로 보기: `.head()`, `.tail()`,`.sample()`

- **`.head(n)`**: 맨 위 n개 (기본값 5) -> "헤더가 잘 들어갔나?" 확인할 때.
- **`.tail(n)`**: 맨 아래 n개 -> "데이터가 끝까지 잘 들어왔나?" 확인할 때.
- **`.sample(n)`**: 무작위 n개 -> **"데이터가 골고루 섞여 있나?"** 편향 없이 확인할 때.

```python
df.head()    # 맨 위 5줄 (기본값)
df.head(3)   # 맨 위 3줄

df.tail()    # 맨 아래 5줄

df.sample(3) # 무작위 3개 뽑기
f.sample(frac=0.5) # 전체 데이터의 50%를 무작위로 뽑기
df.sample(3, random_state=42) # 시드(Seed) 고정: 언제 실행해도 똑같은 애들이 뽑힘
```

####  **💡 Spark와의 차이점**

- **Pandas:** `sample(n=3)` 처럼 **개수**로 뽑는 게 기본입니다.
- **Spark:** `sample(fraction=0.1)` 처럼 **비율**로 뽑는 게 기본입니다. (개수로 뽑으려면 `takeSample`을 써야 함)


### ③ 뼈대와 결측치 확인: `.info()` ⭐ (가장 중요)

데이터의 **'주민등록초본'** 을 떼보는 것과 같습니다.

- **확인할 것:**
    1. **Non-Null Count:** 결측치(NaN)가 있는가? (전체 개수보다 적으면 구멍 난 것!)
    2. **Dtype:** 숫자가 문자로 잘못 잡히지 않았는가? (`object` vs `int/float`)
    3. **Memory Usage:** 메모리를 얼마나 잡아먹나?
    4. **Column**: 컬럼 이름 확인 

```python
df.info()
# <class 'pandas.core.frame.DataFrame'>
# RangeIndex: 1000 entries, 0 to 999
# Data columns (total 5 columns):
#  #   Column   Non-Null Count  Dtype  
# ---  ------   --------------  -----  
#  0   Name     1000 non-null   object 
#  1   Age      990 non-null    float64  <-- 10개 비었네? (결측치)
#  2   Salary   1000 non-null   int64
```

### ④ 통계 요약: `.describe()`

수치형 데이터의 분포를 한눈에 봅니다. (평균, 표준편차, 최소/최대값)

- **용도:** "나이가 200살? 월급이 마이너스?" 같은 **이상치(Outlier)** 를 찾을 때 씁니다.

```python
# 1. 숫자 컬럼만 요약 (기본)
print(df.describe())

# 2. 모든 컬럼(문자 포함) 요약
print(df.describe(include='all'))
# -> 문자는 unique(고유값 개수), top(가장 흔한 값), freq(빈도)가 나옴
```

#### 주의: 스파크랑 문법이 다름!

- **Spark:** `df.describe("Age")` (괄호 안에 이름 넣음)
- **Pandas:** `df[["Age"]].describe()` (먼저 선택하고 함수 씀)
- 판다스 `describe()` 괄호 안에 컬럼명을 넣으면 에러가 납니다!

---
## [비교] Pandas vs Spark ️

같은 기능을 수행하지만, 스파크는 **대용량 분산 처리**라서 방식이 조금 다릅니다.

| **기능**     | **Pandas 🐼**                        | **Spark ⚡️**                                                      | **비고**                    |
| ---------- | ------------------------------------ | ----------------------------------------------------------------- | ------------------------- |
| **행/열 개수** | `df.shape`<br><br>(속성, 0.001초 컷)     | `(df.count(), len(df.columns))`<br><br>(`count`는 Action이라 오래 걸림!) | 스파크에선 `shape`가 없음.        |
| **눈으로 보기** | `df.head()`<br><br>(예쁜 HTML 표)       | `df.show()`<br><br>(ASCII 아트 텍스트)                                 | 스파크 `head()`는 리스트를 반환함.   |
| **구조 확인**  | `df.info()`  <br><br>(Null, 타입, 메모리) | `df.printSchema()`<br><br>(트리 구조로 타입만 보여줌)                        | 스파크는 Null 확인하려면 별도 쿼리 필요. |
| **통계 확인**  | `df.describe()`                      | `df.describe().show()`                                            | 사용법 거의 동일.                |


>"판다스 초보들이 제일 많이 틀리는 게 뭔지 알아? 
>바로 **`df.shape()`** 라고 괄호를 붙이는 거야. 
> `shape`는 데이터가 가진 **'모양' 그 자체(명사)** 라서 괄호가 없고, `head()`나 `describe()`는 무언가 보여달라고 **'행동'하는 것(동사)** 이라 괄호가 있다고 외우면 편해!"