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
---
## 개념 한 줄 요약

**"데이터를 같은 종류끼리 '묶고(Group)', 계산 결과를 '새로운 이름(Named Agg)'으로 만들어내는 판다스의 가장 강력한 기능"**

---
## 동작 원리 (Split-Apply-Combine)

`groupby`는 내부적으로 3단계 과정을 거쳐 데이터를 요리합니다.

1. **Split (나누기):** `category` 같은 기준 열(Key)을 잡고 데이터를 조각조각 나눕니다.
2. **Apply (적용하기):** 각 조각마다 함수(`sum`, `count`, `mean`)를 적용해 계산합니다.
3. **Combine (합치기):** 계산된 결과들을 다시 하나의 표(DataFrame)로 합칩니다.

---
## 핵심 문법: `agg` (이름 짓기와 계산을 동시에!) 

가장 최신의, 그리고 가장 추천하는 **표준 문법**입니다.
**"어떤 컬럼을, 어떻게 계산해서, 무슨 이름으로 저장할지"** 한 눈에 보입니다.

### 문법 공식 (Syntax)

```python
 df.groupby('묶을_기준_컬럼').agg(
     새로운_컬럼_이름=('계산할_대상_컬럼', '사용할_함수')
 )
```

### 실전 코드 (대시보드 예제)

```python
# 1. 'category' 별로 묶어서
grouped_df = df.groupby('category').agg(
    
    # "order_id를 세어서(count) -> '주문건수'라고 부르자"
    주문건수=('order_id', 'count'),
    
    # "total_amount를 합쳐서(sum) -> '총매출액'이라고 부르자"
    총매출액=('total_amount', 'sum'),
    
    # "price를 평균내서(mean) -> '평균단가'라고 부르자"
    평균단가=('price', 'mean')
)

# 결과 확인
print(grouped_df)
```

- **장점:** 계산이 끝난 뒤에 `rename`을 또 할 필요가 없어 코드가 훨씬 간결해집니다.

---
## 마무리: `reset_index()`는 필수!

`groupby`를 하고 나면, 우리가 묶었던 기준(`category`)이 **행 번호(Index)** 로 숨어버립니다. 
차트를 그리거나 표로 보여주려면 다시 **일반 컬럼**으로 꺼내줘야 합니다.

```python
# 인덱스로 들어간 'category'를 다시 밖으로 끄집어냄
final_df = grouped_df.reset_index()

# 이제 Streamlit에 바로 넣을 수 있는 완벽한 표가 됩니다!
st.dataframe(final_df)
```

---
## 실전 팁 (Tip)

- **함수 이름은 문자열로:** `'sum'`, `'mean'`, `'max'`, `'min'`, `'count'` 등은 따옴표로 감싸서 넣으세요.
- **여러 컬럼으로 묶기:** `groupby(['category', 'date'])` 처럼 리스트로 넣으면, **"카테고리별 + 날짜별"** 통계를 한 번에 낼 수 있습니다.
- **사용자 정의 함수:** 만약 내가 만든 함수(`my_func`)를 쓴다면 따옴표 없이 그냥 넣으세요. `agg(내점수=('score', my_func))`