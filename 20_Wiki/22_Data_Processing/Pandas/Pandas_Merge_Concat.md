---
aliases:
  - Pandas Join
  - Merge
  - Concat
  - 데이터병합
  - 테이블결합
tags:
  - Pandas
  - Python
  - Preprocessing
related:
  - "[[Spark_DataFrame_Joins]]"
  - "[[SQL_JOIN_Concept]]"
linked:
  - file:////Users/gong/gong_study_de/pandas/notebooks/merge.ipynb
---
##  개념 한 줄 요약 

* **`pd.concat`**: **"물리적인 결합"**. 접착제처럼 위나 옆으로 그냥 갖다 붙임. (Union)
* **`pd.merge`**: **"논리적인 결합"**. 공통된 열(ID 등)을 찾아서 짝을 맞춰줌. (SQL Join)

---
##  단순히 이어 붙이기 (`pd.concat`) 

데이터의 내용(ID 매칭 등)은 신경 안 쓰고, 그냥 위아래나 양옆으로 합칩니다.
**문법:** `{python}pd.concat([데이터프레임_리스트], axis=방향, ignore_index=True/False)`
- **`[df1, df2]`**: 합칠 데이터들을 반드시 **리스트(대괄호)** 로 묶어서 넣어야 합니다. (가장 중요! )
- **`axis=0`**: 위아래로 쌓기 (행 결합, 기본값)
- **`axis=1`**: 옆으로 붙이기 (열 결합)
- **`ignore_index`**: 기존 인덱스를 무시하고 0부터 다시 멜지 여부.

![[Pasted image 20260131225051.png|400]]

### ① 위아래로 쌓기 (`axis=0`)

상황 : **"1월 데이터 밑에 2월 데이터를 붙여서 1분기 데이터를 만들자!"**

```python
# 리스트 안에 합칠 데이터프레임을 콤마로 구분해서 넣음
# ignore_index=True: 인덱스를 0, 1, 2... 순서대로 깔끔하게 재설정
result = pd.concat([df_jan, df_feb], axis=0, ignore_index=True)
```

### ② 옆으로 붙이기 (`axis=1`)

"X 데이터(Feature) 옆에 정답지(Label)를 딱풀로 붙이자!"

```python
# 두 데이터를 좌우로 나란히 붙임
result = pd.concat([df_features, df_labels], axis=1)
```

**Spark 비교:**
- `pd.concat(axis=0)` 👉 Spark의 **`df.union()`** 과 같습니다.
- `pd.concat(axis=1)` 👉 Spark에는 없음 (Row ID가 보장되지 않아서 위험함).


---
## 스마트하게 조인하기 (`pd.merge`) 

SQL의 **JOIN**과 완전히 똑같습니다. 양쪽 테이블에 있는 **공통된 열(Key)** 을 기준으로 데이터를 합칩니다.

- **문법:** `{python}pd.merge(왼쪽, 오른쪽, on='기준컬럼', how='조인방식')`

### ① 조인 방식 4대장 (`how`)

어떤 데이터를 살릴지 결정하는 옵션입니다.

| 방식 (`how`) | 설명 (비유) | 결과 데이터의 특징 |
| :--- | :--- | :--- |
| **`inner`** (기본값) | **교집합** ∩ | **양쪽 모두에** 키(Key)가 있는 행만 남깁니다. (짝이 없으면 삭제됨) |
| **`left`** | **왼쪽 기준** ⬅️ | **왼쪽 데이터는 다 살리고**, 오른쪽 정보를 붙입니다. (오른쪽에 없으면 `NaN`) |
| **`right`** | **오른쪽 기준** ➡️ | **오른쪽 데이터는 다 살리고**, 왼쪽 정보를 붙입니다. (왼쪽에 없으면 `NaN`) |
| **`outer`** | **합집합** ∪ | 짝이 있든 없든 **전부 다 가져옵니다**. 빈칸은 모두 `NaN`으로 채움. |

### ② 실전 코드 템플릿 (Copy & Paste) 

`left_df`와 `right_df`가 있고, 둘 다 공통 컬럼(예: `id`)을 가지고 있다고 가정합니다.

```python
# 1. Inner Join (교집합) - 가장 안전함 ✅
# 설명: 두 테이블 모두에 존재하는 데이터만 남깁니다.
merged_inner = pd.merge(left_df, right_df, on='id', how='inner')

# 2. Left Join (왼쪽 기준) - 실무 최다 사용 ⭐️
# 설명: 왼쪽 원본은 유지하고, 오른쪽에서 정보를 가져와 붙입니다. (매칭 안 되면 NaN)
merged_left = pd.merge(left_df, right_df, on='id', how='left')

# 3. Right Join (오른쪽 기준)
# 설명: 오른쪽 원본을 유지하고, 왼쪽 정보를 붙입니다. (매칭 안 되면 NaN)
merged_right = pd.merge(left_df, right_df, on='id', how='right')

# 4. Outer Join (합집합) - 전체 데이터 확인용
# 설명: 누락 없이 모든 데이터를 보고 싶을 때 사용합니다.
merged_outer = pd.merge(left_df, right_df, on='id', how='outer')
```

### ③ 컬럼 이름이 다를 때 (`left_on`, `right_on`)

실무에서는 기준 컬럼 이름이 서로 다른 경우가 많습니다. (예: `user_id` vs `id`)

```python
# 왼쪽은 'user_id', 오른쪽은 'id'를 기준으로 합치기
pd.merge(left_df, right_df, 
         left_on='user_id',   # 왼쪽 테이블의 컬럼명
         right_on='id',       # 오른쪽 테이블의 컬럼명
         how='left')
```


---
## 키 값이 중복될 때 (1:N 조인) 🤯

내가 가장 많이 헷갈리는 상황
왼쪽(Customer)에는 ID가 1개뿐인데, 오른쪽(Order)에는 ID가 여러 개(1번 고객이 2번 주문) 있을 때 어떻게 될까요?

**결론:** **"행이 늘어납니다!" (Row Explosion)**
판다스는 오른쪽에서 매칭되는 게 2개면, **왼쪽 데이터도 똑같이 2개로 복사**해서 짝을 맞춰줍니다.

### ① 데이터 준비

```python
customers = pd.DataFrame({
    'id': [1, 2, 3],
    'name': ['Kim', 'Lee', 'Park']
})

orders = pd.DataFrame({
    'id': [1, 1, 3],
    'product': ['Apple', 'Banana', 'Cherry']
})
```
### ② Inner Join 결과 (교집합 + 증식)

ID가 양쪽에 다 있는 1번(Kim)과 3번(Park)만 남습니다. 
이때, **1번 Kim은 주문을 2번 했으므로 2줄로 늘어납니다.**

```python
pd.merge(customers, orders, on='id', how='inner')
```

**결과 확인:** (2번 Lee는 주문 내역 없어서 삭제됨)

|**id**|**name**|**product**|**설명**|
|---|---|---|---|
|**1**|**Kim**|Apple|1번 Kim 데이터 복사됨 (1)|
|**1**|**Kim**|Banana|1번 Kim 데이터 복사됨 (2)|
|3|Park|Cherry|1:1 매칭|

### ③ Left Join 결과 (왼쪽 유지 + 증식) 

왼쪽(Customer)은 무조건 살립니다. 1번 Kim은 주문이 많아서 늘어나고, 2번 Lee는 주문이 없어도 살아남습니다.

```python
pd.merge(customers, orders, on='id', how='left')
```

**결과 확인:** (2번 Lee 생존, Product는 NaN)

| **id** | **name** | **product** | **설명**                      |
| ------ | -------- | ----------- | --------------------------- |
| **1**  | **Kim**  | Apple       | 1번 Kim 복사됨                  |
| **1**  | **Kim**  | Banana      | 1번 Kim 복사됨                  |
| **2**  | **Lee**  | **NaN**     | **주문 안 했지만 Left Join이라 생존** |
| 3      | Park     | Cherry      | 매칭됨                         |