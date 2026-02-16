---
aliases:
  - loc
  - iloc
  - columns
tags:
  - Pandas
related:
  - "[[00_Pandas_HomePage]]"
  - "[[Pandas_Datatypes_Filtering]]"
---
## 개념 한 줄 요약

**"엑셀에서 마우스로 드래그하듯, 데이터프레임에서 내가 원하는 '행(Row)'과 '열(Column)'만 콕 집어내는 기술"**

---
## 컬럼 선택 (Column Selection)

가장 기본이 되는 기능으로, 열의 **이름(Name)** 을 사용해 데이터를 가져옵니다.

### ① 하나의 열만 선택 (`Series` 반환)

```python
# 'price'라는 이름의 컬럼 하나만 가져옵니다.
df['price']
```

### ② 여러 열을 동시에 선택 (`DataFrame` 반환)

**대괄호를 두 번 `[[ ]]`** 써야 표(DataFrame) 형태가 유지된다는 점을 꼭 기억하세요!

```python
# 'user_id'와 'total_amount' 두 개의 컬럼을 가져옵니다.
df[['user_id', 'total_amount']]
```

### ③ 컬럼 이름 확인하기 (`.columns`)

데이터에 어떤 항목들이 있는지 명단을 확인할 때 씁니다.

```python
print(df.columns)
# 결과: Index(['order_id', 'user_id', 'price', ...], dtype='object')
```

---
## loc vs iloc 

행과 열을 동시에 제어하려면 이 두 가지를 구분해서 써야 합니다.

### ① `loc` (Label-based) : "이름표" 보고 찾기

눈에 보이는 **인덱스 라벨(Index Label)** 과 **컬럼 이름(Column Name)** 을 그대로 사용합니다.

- **문법:** `df.loc[행_이름, 열_이름]`

```python
# 문법: df.loc[행_이름, 열_이름]

# 인덱스가 'row_1'인 행의 'price' 값을 가져와라
print(df.loc['row_1', 'price'])

# 'A'행부터 'C'행까지, 'name'과 'age' 컬럼만 가져와라
print(df.loc['A':'C', ['name', 'age']])
```

- **특징:** 슬라이싱할 때 **'끝 번호'를 포함**합니다. (`'A':'C'` → A, B, C 모두 포함)

### ② `iloc` (Integer-based) : "번호(순서)" 보고 찾기

컴퓨터가 매기는 **0부터 시작하는 숫자 인덱스(Position)** 를 사용합니다.

```python
# 문법: df.iloc[행_번호, 열_번호]

# 0번째 행의 2번째 열(3번째 칸) 값을 가져와라
print(df.iloc[0, 2])

# 처음부터 5번째 행까지(0~4), 0~1번 열만 가져와라
print(df.iloc[0:5, 0:2])
```

- **특징:** 슬라이싱할 때 **'끝 번호'를 제외**합니다. (`0:5` → 0, 1, 2, 3, 4 까지만)

---
## 실전 팁: 조건부 선택 (Boolean Indexing)

`loc`은 조건문과 결합할 때 더욱 강력해집니다.

```python
# 가격이 10,000원 이상인 데이터의 '상품명'만 보고 싶을 때
high_price_items = df.loc[df['price'] >= 10000, 'product_name']
```

---
## 요약 표 (Comparison)

| **구분**    | **loc (라벨 기준)**          | **iloc (위치 기준)**     |
| --------- | ------------------------ | -------------------- |
| **행 찾기**  | 인덱스 이름 (예: '2024-01-01') | 행 번호 (예: 0)          |
| **열 찾기**  | 컬럼 이름 (예: 'category')    | 열 번호 (예: 3)          |
| **슬라이싱**  | 끝 포함 (`a:c` → a,b,c)     | 끝 제외 (`0:3` → 0,1,2) |
| **주 사용처** | 데이터 분석, 조건부 조회           | 반복문 처리, 단순 조회        |
