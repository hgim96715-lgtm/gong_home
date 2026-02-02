---
aliases:
  - Partition
  - 파티션
  - repartition
  - coalesce
  - getNumPartitions
  - glom
  - DPP
  - DynamicPartitionPruning
tags:
  - Spark
related:
  - "[[Spark_Architecture]]"
  - "[[Spark_General_Transformations]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Dynamic_Partition_Pruning(DPP)]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step19.ipynb
---
## 개념 한 줄 요약 

**"거대한 피자(데이터)를 여러 명이 동시에 먹기 위해 조각(Partition)낸 것."**

* **스파크의 철학:** "데이터를 쪼개서(Partition) 여러 노드(Executor)에게 나눠주고 동시에 처리한다."
* **공식:** **1 Partition = 1 Task** (파티션 하나당 스레드 하나가 배정됨)

---
##  왜 파티션이 중요한가요? (Parallelism) 

스파크가 아무리 좋은 슈퍼컴퓨터 위에 있어도, **파티션이 1개라면** 아무 소용이 없습니다.

### ❌ 나쁜 예: 파티션 1개

* **상황:** 100GB 데이터를 처리하는데 파티션이 1개다.
* **결과:** 100개의 CPU 코어 중 **1개만 일하고 99개는 놉니다.** (병렬 처리 실패)

### ✅ 좋은 예: 파티션 100개

* **상황:** 100GB 데이터를 100조각으로 나눴다.
* **결과:** 100개의 CPU 코어가 **동시에 달려들어서 순식간에 끝냅니다.**

---
## 내 파티션 확인하기 (`getNumPartitions`, `glom`) 

말로만 듣지 말고 코드로 직접 확인해봐야 이해가 됩니다.

```python
sc = spark.sparkContext
# 0부터 9까지 데이터 생성 (파티션 개수 지정 가능)
rdd = sc.parallelize(range(10), 2) 

# 1. 파티션 개수만 확인
print(f"파티션 개수: {rdd.getNumPartitions()}") 
# 결과: 2

# 2. 파티션 내부 내용물 확인 (glom 사용!)
# 주의: 데이터가 크면 절대 collect() 하지 마세요. take(1) 등으로 확인.
print(rdd.glom().collect())
# 결과: [[0, 1, 2, 3, 4], [5, 6, 7, 8, 9]]
# -> 1번 파티션이 0~4를, 2번 파티션이 5~9를 가짐.
```

---
###  실습용 데이터 1초 만에 만들기: `spark.range()`

매번 리스트 만들고 `parallelize` 하기 귀찮죠? 스파크에는 **"자동 데이터 생성기"** 가 있습니다.

```python
# 1. 기본 사용법 (0부터 19까지 생성)
df = spark.range(0, 20)
df.show(3)
# +---+
# | id|  <-- 컬럼 이름은 무조건 'id'로 고정됩니다!
# +---+
# |  0|
# |  1|
# |  2|
# +---+

# 2. 파티션 개수까지 한 방에 지정하기 (Start, End, Step, Partition)
# "0 이상 100 미만(0~99) 1씩 증가하는 데이터를 5개 파티션에 담아줘!"
df=spark.range(0,20,1,5)
print("DF partitions:", df.rdd.getNumPartitions())
# 결과: 5 (따로 repartition 안 해도 됩니다!)

# 파티션 내부가 너무 클때는 take사용
df.rdd.glom().take(2)
```

**결과**
```text
[[Row(id=0), Row(id=1), Row(id=2), Row(id=3)],
 [Row(id=4), Row(id=5), Row(id=6), Row(id=7)]]
```

>`glom()`은 **"각 파티션에 있는 데이터를 리스트(`[]`) 하나로 묶어줘"** 라는 명령어입니다.
> 파티션이 5개니까, 당연히 리스트도 **5개**가 나옵니다.

#### 왜 4개씩 담겼을까? (나눗셈의 미학) 

스파크는 공평한 걸 좋아합니다.

- **전체 데이터 개수:** 20개 (0부터 19까지)
- **파티션(방) 개수:** 5개
- **계산:** `20 ÷ 5 = 4`

그래서 **"한 방에 4명씩 들어가!"** 라고 배정한 것입니다.


---
##  파티션 개수 조절하기 (튜닝의 핵심) 

데이터를 처리하다 보면 파티션을 **늘리거나 줄여야 할 때**가 옵니다. 상황에 맞춰 골라 쓰세요.

### ① `repartition(n)`: 무조건 섞기 (Expensive)

데이터를 **완전히 뒤섞어서(Full Shuffle)** 새로운 개수로 재분배합니다

- **언제 쓰나요?**
    - 파티션 수를 **늘려서 병렬성(Parallelism)을 높이고 싶을 때** (CPU가 놀고 있을 때)
    - 데이터가 한쪽에만 쏠린 **Data Skew(불균형)**를 해결해서 고르게 펴고 싶을 때
- **특징:** 비용이 매우 비쌉니다(Expensive) 네트워크를 타고 데이터가 다 이동합니다.

**🔬 [실습 검증] 로그로 확인하는 Shuffle**

```python
 # 파티션 5개 -> 8개로 늘리기 (Shuffle 필수)
 df_repart = df.repartition(8)
 df_repart.explain()
```

 **📜 실행 계획 (Physical Plan)**
 
```text
 == Physical Plan ==
 AdaptiveSparkPlan isFinalPlan=true
 +- Exchange RoundRobinPartitioning(8) ...  <-- 👈 [범인] Exchange가 보이면 셔플 발생!
    +- *(1) Range (0, 20, step=1, splits=5)
```

> * **핵심:** `Exchange`라는 단어가 보이면 **"아, 데이터가 네트워크를 타고 대이동(Shuffle)했구나"** 생각하면 됩니다.

---

### ② `repartitionByRange(n, col)`: 정렬하며 섞기 (Ordered)

특정 컬럼의 **범위(Range)** 를 기준으로 파티션을 나눕니다

- **언제 쓰나요?**
    - 데이터가 숫자, 날짜, 타임스탬프처럼 **순서(Range)** 가 중요할 때
    - 이후에 범위 검색 쿼리(`WHERE age BETWEEN 10 AND 20`)를 자주 할 때 유리합니다
- **특징:** 셔플은 발생하지만, 범위 기준으로 정렬되어 모이기 때문에 특정 작업에 최적화됩니다

---

### ③ `coalesce(n)`: 합치기 (Efficient)

셔플을 하지 않고, **인접한 파티션을 그냥 합칩니다(Merge)**

- **언제 쓰나요?**
    - 파티션 수를 **줄일 때** 무조건 이걸 써야 합니다
    - `filter()` 등으로 데이터를 왕창 걸러내서, 파티션 크기가 너무 작아졌을 때
    - 파일 저장 전에 파티션 수를 줄여서 **파일 개수(Small Files)**를 관리할 때.
- **특징:** 셔플이 없어서 빠르지만, **파티션을 늘릴 수는 없습니다**.

 **🔬 [실습 검증] 로그로 확인하는 No Shuffle**

```python
 # 파티션 5개 -> 2개로 줄이기 (Shuffle 없음)
 df_coal = df.coalesce(2)
 df_coal.explain()
```

**📜 실행 계획 (Physical Plan)**

```text
 == Physical Plan ==
 Coalesce 2  <-- 👈 Exchange가 없음! 그냥 합치기만 함.
 +- *(1) Range (0, 20, step=1, splits=5)
```

> * **핵심:** `Exchange` 노드가 없고 `Coalesce`만 보입니다. 즉, **데이터 이동 없이 칸막이만 치웠다**는 뜻입니다.

---


---
## 한눈에 보는 비교표 (Comparison)

|**특징**|**Repartition**|**RepartitionByRange**|**Coalesce**|
|---|---|---|---|
|**셔플 (Shuffle)**|✅ **발생 (Full Shuffle)**|✅ **발생 (Range Shuffle)**|❌ **발생 안 함 (No Shuffle)**|
|**파티션 늘리기**|✅ 가능|✅ 가능|❌ **불가능**|
|**파티션 줄이기**|✅ 가능|✅ 가능|✅ **가능 (추천)**|
|**비용 (Cost)**|💰 **비쌈 (High)**|💰 **비쌈 (High)**|⚡️ **저렴 (Low)**|
|**사용처**|병렬성 증대, 스큐 해결|범위 검색 최적화|파일 저장 전, 필터링 후|

**💡 비유 (Analogy)**

- **Repartition:** "전학 가기" (모든 학생이 짐 싸서 새로운 반으로 배정됨 - 대공사)
- **Coalesce:** "칸막이 치우기" (1반과 2반 사이의 벽만 허물어서 합반함 - 쉬움)

---
## 똑똑하게 읽기: Partition Pruning 

데이터를 다 읽고 나서 거르는(`filter`) 건 하수입니다. **처음부터 필요한 파티션만 읽는 것**이 고수입니다.

### ① Static Partition Pruning (정적 가지치기)

쿼리를 짤 때 **조건이 명확한 경우**입니다.

- **상황:** `SELECT * FROM sales WHERE region = 'US'`
- **동작:** 스파크가 컴파일 타임에 "아, 'US' 폴더만 읽으면 되겠네?" 하고 나머지는 쳐다도 안 봅니다.

### ② Dynamic Partition Pruning (DPP, 동적 가지치기)

**스파크 3.0**의 핵심 기능입니다. 조건이 **다른 테이블과의 조인(Join)** 결과에 따라 달라질 때 사용합니다.

- **상황:** "Sales(대용량) 테이블에서, Region(소용량) 테이블에 있는 'North America' 지역만 가져와라."
- **동작:** 작은 테이블을 먼저 읽어 조건을 알아낸 뒤(`North America`), 큰 테이블을 읽기 전에 **미리 파티션을 걸러냅니다(Pruning).**
- **효과:** I/O 비용 급감, 조인 속도 향상.

---
##  [실험] 파티션 개수와 성능의 진실

"파티션이 많으면 무조건 빠를까요?" 직접 코드로 돌려보면 **"아니다"** 라는 것을 알 수 있다!
단순한 작업(`count`)을 할 때, 불필요한 `repartition`이 얼마나 큰 비용(Shuffle)을 발생시키는지 확인해 봅시다.

### 실험 코드

```python
import time

# 5천만 건의 데이터 생성 (약 400MB ~ 1GB 수준)
big_df = spark.range(0, 50_000_000)

def partition_duration(df, label):
    start = time.time()
    df.count()  # Action 수행
    end = time.time()
    print(f"{label}: {end - start:.2f} seconds")

# Case 1: 파티션 1개로 합치기 (Shuffle 없음 - Coalesce)
partition_duration(big_df.coalesce(1), "coalesce(1)")

# Case 2: 파티션 8개로 늘리기 (Shuffle 발생 - Repartition)
partition_duration(big_df.repartition(8), "repartition(8)")

# Case 3: 파티션 200개로 과하게 늘리기 (Shuffle + Scheduling Overhead)
partition_duration(big_df.repartition(200), "repartition(200)")
```

### 실험 결과

```text
coalesce(1): 0.71 seconds       <-- 🏆 가장 빠름! (셔플이 없어서)
repartition(8): 2.02 seconds    <-- 🐢 느려짐 (셔플 비용 발생)
repartition(200): 2.11 seconds  <-- 🐢 더 느려짐 (셔플 + 태스크 관리 오버헤드)
```

### 결과 해석 (Why?) 🤔

1. **`coalesce(1)` 승리:** 데이터가 아주 크지 않고 단순 집계(`count`)만 할 때는, 네트워크를 타는 **셔플(Shuffle)** 비용이 병렬 처리 이득보다 큽니다. `coalesce`는 셔플을 안 해서 빠른 겁니다.(데이터가 아주 크지 않고 단순 집계일 때 효율적)

2. **`repartition`의 함정:** 파티션을 늘려서 병렬성을 얻으려 했지만, **데이터를 재분배하는 비용(Shuffle)** 이 더 들어서 오히려 느려졌습니다.

3. **`200`개의 오버헤드:** 파티션이 너무 많으면 스파크 마스터가 200개의 태스크를 관리하고 띄우는 데 드는 시간(Overhead)이 추가되어 더 느려집니다.

> **결론:** 무작정 `repartition`을 쓰지 말고, **셔플 비용 vs 병렬 처리 이득**을 잘 따져봐야 합니다. 
> 단순히 파티션을 줄일 때는 셔플 없는 **`coalesce`** 가 압도적으로 유리합니다!
