---
aliases:
  - GroupBy
  - Agg
  - Sort
  - OrderBy
  - 집계
  - 정렬
  - 순위
tags:
  - Spark
related:
  - "[[DataFrame_Transform_Basic]]"
  - "[[Spark_DataFrame_Basics]]"
  - "[[SQL_with_Spark]]"
  - "[[00_Apache_Spark_HomePage]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step10.ipynb
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step8.ipynb
---

## 1개념 한 줄 요약 

**"데이터를 끼리끼리 묶어서(GroupBy) 계산하고(Agg), 순서대로 줄 세우는(Sort) 분석 과정."**

* **1부([[DataFrame_Transform_Basic]])** 가 데이터를 다듬는 과정이라면,
* **2부(이 파일)** 는 다듬어진 데이터로 **통계와 인사이트**를 뽑는 과정입니다.

---
##  집계하기 (`groupBy` + `agg`) 

**"묶고(`groupBy`) -> 계산한다(`agg`)"** 는 공식처럼 함께 다닙니다.

### ① `agg`가 대체 뭔가요?

`groupBy("dept")`만 하면 스파크는 계산을 안 하고 대기합니다.
그냥 "아, 묶어야 하는구나" 하고 대기 상태(**GroupedData** 객체)로 있습니다. 
데이터프레임이 아니기 때문에 눈으로 볼 수도 없습니다.

**`agg`** 는 대기 중인 그룹들을 **실제로 계산(Sum, Avg)하고, 다시 데이터프레임으로 만들어주는 역할**을 합니다.

### ② 기본 구조 (단일 계산)

```python
# 부서별(dept) 평균 월급(salary) 구하기
# groupBy가 묶고 -> agg가 계산해서 -> DataFrame으로 만듦
df.groupBy("dept").agg( F.avg("salary") ).show()
```

### ③  여러 통계 한 번에 구하기 (Best Practice) 

`agg` 안에 여러 함수를 넣어서 한 번에 계산합니다.
이때 **`.alias()`** 를 안 쓰면 컬럼명이 `avg(salary)` 같이 지저분해지므로 필수입니다!

```python
import pyspark.sql.functions as F

df.groupBy("dept").agg(
    F.count("*").alias("emp_count"),       # 직원 수
    F.sum("salary").alias("total_salary"), # 총 급여
    F.avg("salary").alias("avg_salary"),   # 평균 급여
    F.max("salary").alias("max_salary")    # 최고 급여
).show()
```

### ④  여러 컬럼으로 묶기 (Multi-Grouping) 

"카테고리별로 묶고, 그 안에서 상태별로 또 쪼개라!"
**"A이면서 동시에 B인 애들끼리 묶어라!"** 라는 뜻입니다.

```python
# "카테고리(category)와 배송상태(status)가 똑같은 애들끼리 묶어서 세어봐!"
df.groupBy("category", "status").agg( 
	F.count("*").alias("order_count") 
).orderBy("category", "status").show()
```

- **해석:** (전자제품, 배송중), (전자제품, 배송완료), (가구, 배송중)... 이렇게 모든 조합별로 카운트를 셉니다.

>이렇게 여러 개로 묶었을 때는 결과가 뒤죽박죽 나오기 쉬우므로,
> 마지막에 꼭 **`.orderBy("category", "status")`** 를 붙여줘야 보기 좋게 정렬됩니다!

### ⑤ GroupedData의 비밀 (Shortcut vs Agg)

`groupBy("col")`을 실행한 직후는 DataFrame이 아니라 **`GroupedData`** 라는 특수한 객체입니다.
이 객체는 자주 쓰는 집계를 위한 **지름길(Shortcut)** 메서드를 제공합니다.

| 방식           | 코드                                | 장점                            | 단점                        |
| :----------- | :-------------------------------- | :---------------------------- | :------------------------ |
| **지름길**      | `.count()`, `.sum("salary")`      | 코드가 짧고 직관적임                   | 컬럼명 변경 불가, 한 번에 하나만 계산 가능 |
| **정석 (agg)** | `.agg(F.count("*").alias("cnt"))` | **이름 변경 가능**, **여러 통계 동시 계산** | 코드가 조금 길어짐                |

> ** 조언:**
> "간단히 개수만 확인할 땐 `.count()`를 써도 되지만.
> 하지만 실무에서는 리포트를 만들어야 하니까, **이름(`alias`)을 예쁘게 붙일 수 있는 `.agg()`를 쓰는 습관**을 들이는 게 좋아!"


---

## 정렬하기 (`orderBy` / `sort`) 

데이터를 순서대로 줄 세웁니다.
**`orderBy`와 `sort`는 100% 똑같은 쌍둥이 함수입니다.** (편한 거 쓰세요!)

### ① 기본 정렬 (오름차순/내림차순)

```python
# 1. 오름차순 (기본값)
df.orderBy("age").show()

# [방법 2] sort 사용 (Pandas 스타일) - 결과 똑같음!
df.sort("age").show()

# 2. 내림차순 (.desc()) - 반드시 col 객체 필요!, 오름차순도!
df.orderBy(F.col("age").desc()).show()
# 또는 
df.sort(F.col("age").desc()).show()

# [Tip] 함수 스타일로도 가능
df.orderBy(F.desc("age")).show()
```

### ② 여러 기준 정렬 (다중 정렬) 

**"1차 기준으로 먼저 크게 줄을 세우고, 그 안에서(같은 값끼리만) 2차 기준으로 다시 줄을 세웁니다."**

```python
df.orderBy(
    F.col("dept").asc(),  ## 1차: 먼저 부서별로 모은다. (Accounting -> HR -> IT ...)
    F.col("age").desc()   # 2차: 같은 부서 사람들 안에서, 나이 많은 순으로 다시 정렬!
).show()
```

> **Tip:** `IT`팀이 먼저 나오고, `IT`팀 안에서는 나이 많은 김부장이 이대리보다 위에 나옵니다.

### ③  결측치(Null) 위치 제어하기 

스파크에서 정렬하면 기본적으로 **Null이 맨 앞(오름차순)이나 맨 뒤(내림차순)** 에 나옵니다. 
이게 보기 싫다면 **위치를 강제로 지정**해야 합니다.

```python
# 나이 오름차순 하되, Null은 맨 마지막으로 보내라! (가장 많이 씀)
df.orderBy(F.col("age").asc_nulls_last()).show()

# 나이 내림차순 하되, Null을 맨 처음으로 올려라!
df.orderBy(F.col("age").desc_nulls_first()).show()
```

### ④ [성능] `sortWithinPartitions` 

전체 데이터의 완벽한 순위는 필요 없고, **"파일 하나하나 내부에서만 정렬되면 돼"** 하는 경우에 씁니다. 
(전체 셔플이 안 일어나서 속도가 훨씬 빠릅니다.)

```python
df.sortWithinPartitions("age").show()
```

---
## 실전 패턴: 집계 후 정렬하기 (Agg -> Sort) 

가장 많이 쓰는 패턴입니다. 
통계를 내고 나서, 그 결과값으로 순위를 매깁니다.

```python
# 1. 카테고리별로 묶어서 개수를 세고 (cnt)
# 2. 개수가 많은 순서대로 정렬해라!
df.groupBy("category")\
  .agg( F.count("*").alias("cnt") )\
  .sort( F.col("cnt").desc() )\
  .show()
```




