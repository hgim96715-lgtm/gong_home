---
aliases:
  - Common Mistakes
  - Syntax Error
  - 실수모음
tags:
  - Spark
  - Troubleshooting
related:
  - "[[Spark_DataFrame_Basics]]"
  - "[[DataFrame_Aggregation]]"
  - "[[Spark_Iterating_Data]]"
  - "[[Spark_Graph_Analysis_Basic]]"
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
