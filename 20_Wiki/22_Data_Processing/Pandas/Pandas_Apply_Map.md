---
aliases:
  - 데이터 변환
  - apply
  - map
  - lambda
  - 사용자정의함수
  - 전처리
tags:
  - Python
  - Pandas
related:
  - "[[Python_Lambda_Map]]"
  - "[[Pandas_Groupby]]"
  - "[[00_Pandas_HomePage]]"
---
## 개념 한 줄 요약

**"데이터프레임의 모든 칸(Cell)이나 줄(Row)을 훑으면서, 내가 만든 '규칙(함수)'을 하나씩 적용해 바꾸는 기술"**

---
## apply 문법 구조 (Syntax)

**⚠️ 주의: 이 기능은 Pandas(`DataFrame`, `Series`)에만 존재합니다!** 
(파이썬 기본 리스트(`[]`)나 딕셔너리에는 `.apply()`가 없습니다.)

가장 기본 뼈대는 **"누구에게(Data) + 적용해라(apply) + 이 규칙을(Function)"** 입니다.

```python
# 기본 공식
데이터.apply(함수)
```

이때 **'데이터'** 가 **한 줄(Column)** 이냐, **표 전체(DataFrame)** 냐에 따라 `x`가 무엇인지가 달라집니다.


---
## `apply` + `lambda` 

가장 많이 쓰는 패턴입니다.
특정 컬럼의 데이터를 하나씩 꺼내서 변형하고 싶을 때 사용합니다.

### ① 한 줄(Column) 변형하기

```python
# 'total_amount' 컬럼의 숫자들을 천 단위 콤마 + '원' 문자로 변경
# x에는 각 행의 값이 하나씩 들어갑니다. (10000 -> "10,000원")
df['total_amount'] = df['total_amount'].apply(lambda x: f"{int(x):,}원")
```

### ② 내 함수 적용하기

로직이 복잡해서 `lambda` 한 줄로 안 될 때는, 미리 함수를 만들어두고 `apply`에 이름만 넣습니다.

```python
def classify_age(age):
    if age < 20: return "청소년"
    elif age < 60: return "성인"
    else: return "시니어"

# 함수 이름(classify_age)만 괄호 없이 넣습니다.
df['age_group'] = df['age'].apply(classify_age)
```

---
## `axis=1` : 여러 컬럼 동시에 쓰기 

하나의 컬럼만 보는 게 아니라, **"같은 행에 있는 옆 컬럼의 값"** 도 같이 참고해서 계산해야 할 때 씁니다. 
이때는 `lambda x`의 `x`가 **값 하나가 아니라 '행 전체(Row)'** 가 됩니다.

```python
# 가격(price) * 수량(qty) = 총액(total) 계산
# axis=1 : "세로(열) 방향으로 이동하지 말고, 가로(행) 방향 데이터를 묶어서 줘라"
df['total'] = df.apply(lambda row: row['price'] * row['qty'], axis=1)
```

---
## `map` : 딕셔너리 매핑 전문가 

복잡한 계산 말고, **"A는 B로 바꿔라"** 같은 단순 1:1 교환에는 `map`이 훨씬 빠르고 직관적입니다.

```python
# 성별 코드를 한글로 변경
gender_dict = {'M': '남자', 'F': '여자'}

# 딕셔너리를 map에 넣으면 알아서 키(Key)를 찾아 값(Value)으로 바꿈
df['gender_kor'] = df['gender'].map(gender_dict)
```

**팁:** 딕셔너리에 없는 값은 `NaN`(결측치)으로 변해버리니 주의하세요!

---
## 요약 표 (Comparison)

|**구분**|**apply**|**map**|
|---|---|---|
|**적용 대상**|DataFrame(행/열 전체), Series(컬럼)|**Series(컬럼)** 전용|
|**주 사용처**|복잡한 로직, 함수 적용, 행 단위 계산|**1:1 치환 (Dictionary Mapping)**|
|**함수 사용**|`apply(함수)` 가능|`map(함수)` 가능|
|**속도**|로직에 따라 다름 (보통)|단순 치환 시 매우 빠름|
