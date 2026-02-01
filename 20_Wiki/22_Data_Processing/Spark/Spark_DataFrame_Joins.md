---
aliases:
  - Spark Join
  - Merge
  - Inner Join
  - Left Join
  - Anti Join
  - Semi Join
  - 데이터병합
tags:
  - Spark
  - JOIN
  - SQL
related:
  - "[[DataFrame_Transform_Basic]]"
  - "[[SQL_with_Spark]]"
  - "[[Pandas_Merge_Concat]]"
  - "[[SQL_JOIN_Concept]]"
---
## Join의 기본 문법 

**"두 개의 데이터프레임(좌, 우)을 공통된 열(Key)을 기준으로 합치자."**

```python
# 기본 문법
joined_df = left_df.join(right_df, on='기준컬럼', how="조인타입")
```

- **`on`**: 기준이 되는 컬럼명 (예: `'id'` 또는 `['id', 'date']`)
-  **`how`**: 조인 방식 (`inner`, `left`, `full`, `leftsemi`, `leftanti`)

---
## Join 타입 총정리 (6가지) 🎨

| **타입 (how)**       | **설명 (비유)**       | **결과 데이터**                                        |
| ------------------ | ----------------- | ------------------------------------------------- |
| **`inner`**        | **교집합** ∩         | 양쪽 모두에 아이디가 있는 사람만 남음. (완벽한 커플)                   |
| **`full`** (outer) | **합집합** ∪         | 짝이 있든 없든 전부 다 나옴. 없으면 `null`로 채움.                 |
| **`left`**         | **왼쪽 기준** ⬅️      | 왼쪽(User)은 다 살리고, 오른쪽(Salary)은 있으면 붙이고 없으면 `null`. |
| **`right`**        | **오른쪽 기준** ➡️     | 오른쪽(Salary)은 다 살리고, 왼쪽(User)은 매칭되는 것만.            |
| **`left_semi`**    | **존재 확인 (필터)** 🔍 | "오른쪽에 **있는** 애들만 남겨줘." (데이터는 왼쪽 것만 가짐)            |
| **`left_anti`**    | **차집합 (필터)** 🚫   | "오른쪽에 **없는** 애들만 남겨줘." (데이터는 왼쪽 것만 가짐)            |
![[Pasted image 20260131223034.png|500x500]]


---
## 3. 실전 코드 예제 (Generic Template) 

`left_df`와 `right_df`가 있고, 둘 다 공통 컬럼 `id`를 가지고 있다고 가정합니다.

### ① 기본 4대장 (Inner, Full, Left, Right)

가장 많이 사용하는 조인 방식입니다. 데이터가 합쳐져서 컬럼이 늘어납니다


```python
# 1. Inner Join (교집합) ∩
# 설명: "양쪽 모두에 'id'가 존재하는 행만 남깁니다." (가장 엄격함)
# 결과: 짝이 없는 데이터는 모두 사라짐.
left_df.join(right_df, on="id", how="inner").show()

# 2. Full Outer Join (합집합) ∪
# 설명: "짝이 있든 없든 무조건 다 가져옵니다."
# 결과: 매칭 안 되는 곳은 null(결측치)로 채워짐.
left_df.join(right_df, on="id", how="full").show()

# 3. Left Join (왼쪽 기준) ⬅️ [실무 최다 사용]
# 설명: "왼쪽 데이터는 무조건 다 살리고, 오른쪽 정보를 붙입니다."
# 결과: 오른쪽에 정보가 없으면 null이 붙음. (왼쪽 행 개수 유지됨)
left_df.join(right_df, on="id", how="left").show()

# 4. Right Join (오른쪽 기준) ➡️
# 설명: "오른쪽 데이터는 무조건 다 살리고, 왼쪽 정보를 붙입니다."
# 결과: 왼쪽에 정보가 없으면 null이 붙음.
left_df.join(right_df, on="id", how="right").show()
```

### ② 필터링용 조인 (Semi & Anti) ⭐️

**"Join의 탈을 쓴 Filter"** 입니다. 
오른쪽 테이블(`right_df`)의 데이터를 가져오지 않고, **오직 왼쪽 테이블(`left_df`)의 행을 살릴지 죽일지만 결정**합니다.


```python
# 5. Left Semi Join (존재 확인) 🔍
# 설명: "오른쪽(참조) 테이블에 id가 '있는' 것만 남겨라."
# 특징: 오른쪽 테이블의 컬럼은 추가되지 않음! (Inner Join과의 차이점)
# 용도: "구매 이력이 있는(Right) 회원(Left)만 뽑고 싶을 때"

left_df.join(right_df, on="id", how="leftsemi").show()

# 6. Left Anti Join (차집합) 🚫
# 설명: "오른쪽(참조) 테이블에 id가 '없는' 것만 남겨라."
# 특징: SQL의 'NOT IN'과 동일한 효과. 성능이 훨씬 좋음.
# 용도: "구매 이력이 없는(Right) 유령 회원(Left)을 찾고 싶을 때"

left_df.join(right_df, on="id", how="leftanti").show()
```

---
## [주의] 중복 컬럼 방지 꿀팁 (`on=["col"]`) 

조인할 때 `df1.id == df2.id` 조건을 쓰면, 결과에 `id` 컬럼이 두 개가 생겨서 나중에 에러가 납니다.
가장 많이 겪는 `Reference 'id' is ambiguous` 에러 해결법입니다.
자동병합으로 깔끔하게 해야지 에러가 안남 

```python
# ❌ 나쁜 예 (Ambiguous Error 발생 위험)
# 결과: id(user), id(salary), name, salary ... (id가 두 개!)
df.join(other_df, df.id == other_df.id, "inner")

# ⭕️ 좋은 예 (자동 병합)
# 결과: id, name, salary ... (id가 하나로 깔끔하게 합쳐짐)
# 조건: 양쪽 테이블의 컬럼 이름("id")이 같아야 함.
df.join(other_df, on=["id"], how="inner")
```
