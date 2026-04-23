---
aliases:
  - 그룹화
  - 집계
  - agg
  - 피벗테이블
  - group by
  - apply
tags:
  - Pandas
related:
  - "[[Pandas_Selection]]"
  - "[[00_Pandas_HomePage]]"
  - "[[Pandas_Apply_Map]]"
  - "[[Pandas_Pivot]]"
  - "[[Pandas_Inspection]]"
---
# Pandas_Groupby

> **"데이터를 같은 종류끼리 묶고(Group), 계산 결과를 새로운 이름으로 만들어내는 판다스의 가장 강력한 기능"**

---

## 동작 원리 (Split → Apply → Combine)

`groupby`는 내부적으로 3단계로 동작해요.

```
① Split   : 기준 컬럼(Key)으로 데이터를 조각조각 나눔
② Apply   : 각 조각마다 함수(sum, count, mean) 적용
③ Combine : 결과들을 다시 하나의 DataFrame으로 합침
```

---

## 핵심 문법: `agg` (Named Aggregation)

가장 최신의, 가장 추천하는 표준 문법이에요. "어떤 컬럼을, 어떻게 계산해서, 무슨 이름으로 저장할지" 한눈에 보여요.

### 문법 공식

```python
df.groupby('묶을_기준_컬럼').agg(
    새로운_컬럼_이름=('계산할_대상_컬럼', '사용할_함수')
)
```

### 실전 코드

```python
grouped_df = df.groupby('category').agg(

    # "order_id를 세어서(count) → '주문건수'라고 부르자"
    주문건수=('order_id',     'count'),

    # "total_amount를 합쳐서(sum) → '총매출액'이라고 부르자"
    총매출액=('total_amount', 'sum'),

    # "price를 평균내서(mean) → '평균단가'라고 부르자"
    평균단가=('price',        'mean')
)

print(grouped_df)
```

> 장점: 계산 후 `rename`을 따로 할 필요 없어서 코드가 훨씬 간결해요.

---

## 마무리: `reset_index()` 필수!

`groupby` 후 기준 컬럼이 **인덱스로 숨어버려요.** 차트나 표로 쓰려면 다시 꺼내줘야 해요.

```python
# 인덱스로 숨어버린 'category'를 다시 일반 컬럼으로
final_df = grouped_df.reset_index()

# 이제 Streamlit에 바로 넣을 수 있음
# st.dataframe(final_df)
```

---

## 자주 쓰는 집계 함수

|함수|의미|예시|
|---|---|---|
|`'count'`|개수 세기 (NaN 제외)|주문 건수|
|`'sum'`|합계|총 매출|
|`'mean'`|평균|평균 단가|
|`'max'`|최댓값|최고 점수|
|`'min'`|최솟값|최저가|
|`'std'`|표준편차|변동성|
|`'nunique'`|고유값 개수|방문한 사용자 수|

---

## 실전 팁

```python
# ① 함수 이름은 문자열로 (따옴표 필수)
agg(건수=('id', 'count'))   # ✅
agg(건수=('id', count))     # ❌ 에러

# ② 여러 컬럼으로 묶기
df.groupby(['category', 'date']).agg(...)
# "카테고리별 + 날짜별" 통계를 한번에

# ③ 사용자 정의 함수 (따옴표 없이)
def my_func(x):
    return x.max() - x.min()

df.groupby('category').agg(범위=('price', my_func))

# ④ count() 단독 사용 (세분류별 개수 확인)
df.groupby("세분류코드명")["컬럼명"].count()
```

---

## transform vs agg 차이

```python
# agg: 그룹별 1개 값으로 압축 (행 수 줄어듦)
df.groupby('category')['price'].agg('mean')
# category A → 100
# category B → 200

# transform: 원본과 같은 행 수 유지 (각 행에 그룹 통계 붙이기)
df['category_mean'] = df.groupby('category')['price'].transform('mean')
# 각 행에 자기 그룹의 평균이 붙음
```

---
