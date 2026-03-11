---
aliases:
  - Common Mistakes
  - Syntax Error
  - 실수모음
tags:
  - Spark
  - Troubleshooting
related:
  - "[[Spark_DataFrame]]"
  - "[[DataFrame_Aggregation]]"
  - "[[Spark_Iterating_Data]]"
  - "[[Spark_Graph_Analysis_Basic]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 집계(Aggregation) 중 정렬 시도 🚫

**증상:** `AttributeError`가 나거나 코드가 실행되지 않음.

### ❌ 틀린 코드 (Bad)

```python
# 에러! .agg() 괄호 안에서 .desc()를 쓰면 안 됩니다.
df.groupBy("id").agg(
    F.count("item").alias("cnt").desc() 
)
```

### ⭕️ 맞는 코드 (Good)

```python
(
    df.groupBy("id")
    .agg(                     
        F.count("item").alias("cnt")
    )                         # 1. 여기서 괄호 닫고 집계 끝냄!
    .orderBy(F.desc("cnt"))   # 2. 그 다음에 정렬 붙이기
)
```

---
## 값 하나만 꺼내고 싶은데... (`[0][0]`) 

**증상:** `collect()` 했더니 `[Row(age=20)]` 같은 이상한 리스트가 나옴. 그냥 숫자 `20`을 원함.

**해결:** 스파크는 항상 **List of Rows**를 반환합니다. 껍질을 두 번 까야 합니다.

> [[Spark_Data_Cleaning#🧐 분석 왜 `collect()[0][0]` 인가요? (이중 리스트의 비밀)|이중리스트]] 참고 

```python
val = df.select("age").collect()       # [Row(age=20)]
real_val = val[0][0]                   # 20 (성공!)
# 또는 이름으로 접근
real_val = val[0]["age"]               # 20 (더 안전함 ✅)
```
