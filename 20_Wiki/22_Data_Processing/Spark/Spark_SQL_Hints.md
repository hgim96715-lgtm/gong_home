---
aliases:
  - SQL Hint
  - Broadcast Hint
  - Join Hing
  - 쿼리 힌트
tags:
  - Spark
related:
  - "[[Spark_Catalyst_Optimizer]]"
  - "[[Spark_AQE_Deep_Dive]]"
  - "[[00_Apache_Spark_HomePage]]"
---
## 개념 한 줄 요약 

**"스파크의 자동 최적화(CBO)를 믿지 못할 때, 개발자가 직접 실행 계획을 지정해버리는 강제 명령."**

* **원칙:** 스파크의 Catalyst Optimizer는 보통 개발자보다 똑똑합니다. 알아서 최적의 길을 찾습니다.
* **예외:** 하지만 통계 정보가 부정확하거나 복잡한 쿼리에서는 멍청한 선택(예: 큰 테이블을 Broadcast하려 함)을 할 때가 있습니다. 이때 **Hint**를 써서 바로잡습니다. 

---
## 문법 (Syntax) 

SQL과 DataFrame API 두 가지 방식이 있습니다.

### ① SQL 방식 (`/*+ ... */`)

주석처럼 생긴 특수 문법을 `SELECT` 바로 뒤에 붙입니다. 

```sql
-- 문법: SELECT /*+ 힌트이름(테이블명) */ ...
SELECT /*+ BROADCAST(small_table) */ * FROM large_table 
JOIN small_table ON large_table.id = small_table.id;
```

### ② PySpark DataFrame 방식 (`.hint()`)

SQL과 달리 **"힌트를 적용할 DataFrame 객체"**에 직접 `.hint()`를 붙여야 합니다.

#### 1. Broadcast Hint (매개변수 없음)

SQL처럼 테이블 이름을 넣지 않습니다. **작은 테이블 객체**에 직접 붙입니다.

```python
# [정석 패턴] 큰 테이블.join(작은 테이블.hint("broadcast"), 조건)
large_df.join(small_df.hint("broadcast"), "key")

# [주의] 아래처럼 쓰면 'large_df'를 방송하라는 뜻이 되어 OOM이 날 수 있습니다!
# large_df.hint("broadcast").join(small_df, "key")  <-- (X) 위험
```

#### 2. Repartition / Coalesce Hint (매개변수 있음)

파티션 개수(숫자)를 매개변수로 넘겨줍니다.

문법: `{python}df.hint("힌트이름", "매개변수")`

```python
# 파티션을 10개로 줄이기 (Coalesce)
df.hint("coalesce", 10).show()

# 파티션을 100개로 재분배하기 (Repartition)
df.hint("repartition", 100).show()
```

#### 3. Join Strategy Hint (전략 지정)

조인할 때 특정 알고리즘을 강제하고 싶다면 해당 DataFrame에 붙입니다.

```python
# Sort-Merge Join 강제
df1.hint("merge").join(df2, "key")

# Shuffle Hash Join 강제
df1.hint("shuffle_hash").join(df2, "key")
```

---
## 자주 쓰는 힌트 TOP 3 

### ① BROADCAST (조인 최적화)

- **설명:** 한쪽 테이블이 작을 때, **모든 노드에 복제(Broadcast)** 해서 셔플을 없애버립니다.
    
- **언제 씀?**
    - 작은 테이블(Dimension)과 큰 테이블(Fact)을 조인할 때.
    - `spark.sql.autoBroadcastJoinThreshold` 설정보다 살짝 큰 테이블을 강제로 Broadcast 하고 싶을 때.
- **코드:** `/*+ BROADCAST(table_name) */`

### ② MERGE / SHUFFLE_HASH (조인 전략 강제)

- **MERGE:** Sort-Merge Join을 강제합니다. (대용량 조인의 정석)
- **SHUFFLE_HASH:** Shuffle Hash Join을 강제합니다. (Sort 비용을 아끼고 싶을 때, 메모리가 충분하다면 유리)
- **코드:** `/*+ MERGE(table) */`, `/*+ SHUFFLE_HASH(table) */`

### ③ COALESCE / REPARTITION (파티션 제어)

- **설명:** 쿼리 결과의 파티션 수를 강제로 조절합니다.
- **COALESCE:** 파티션을 줄일 때 (셔플 없음).
- **REPARTITION:** 파티션을 늘리거나, 데이터 쏠림(Skew)을 해결하려고 강제로 섞을 때.
- **코드:** `/*+ COALESCE(10) */`, `/*+ REPARTITION(100) */`

---
## 힌트가 무시되는 경우 🚫

힌트를 쓴다고 스파크가 100% 말을 듣는 건 아닙니다. 스파크는 힌트를 **"가능하면 들어주려고 노력"** 하지만, 논리적으로 말이 안 되거나 대상을 못 찾으면 **조용히 무시(Ignore)** 하고 제 갈 길을 갑니다.

### ① 논리적으로 불가능할 때 (Join Type 제약)

조인 종류에 따라 **Broadcast**가 불가능한 방향이 있습니다.
* **상황:** `Right Outer Join`을 하는데 **왼쪽 테이블**에 Broadcast를 걸면 무시됩니다.
* **이유:** Right Join은 오른쪽 테이블이 기준이므로, 왼쪽 테이블을 방송(Broadcast)해서 복제하는 방식이 구조적으로 맞지 않을 수 있습니다.

### ② 테이블 이름 불일치 (Alias 실수) ⚠️ **(가장 흔한 실수)**

쿼리에서 테이블에 **별명(Alias)** 을 붙여놓고, 힌트에는 **원본 테이블 이름**을 쓰면 스파크는 힌트 대상을 찾지 못해 무시합니다.

**❌ [실패 예시] 별명(`d`)을 썼는데 원본 이름(`dim_table`)으로 힌트 줌**

```python
# 테이블 등록
fact_df.createOrReplaceTempView("fact_table")
dim_df.createOrReplaceTempView("dim_table")

# 스파크 입장: "난 'd'라는 테이블은 아는데 'dim_table'은 지금 안 보이는데?" -> 힌트 무시 -> SortMergeJoin 수행
spark.sql("""
    SELECT /*+ BROADCAST(dim_table) */ * FROM fact_table f
    JOIN dim_table d ON f.user_id = d.user_id
""").explain()
```

⭕️ [성공 예시] 별명(`d`)으로 정확히 힌트 줌

```python
# 스파크 입장: "OK, 'd' 테이블을 Broadcast 하라고? 접수 완료!" -> BroadcastHashJoin 수행
spark.sql("""
    SELECT /*+ BROADCAST(d) */ * FROM fact_table f
    JOIN dim_table d ON f.user_id = d.user_id
""").explain()
```

**💡 요약:** `FROM table_name AS alias` 를 썼다면, 힌트에도 반드시 `/*+ BROADCAST(alias) */` 라고 적어야 합니다!


> "Hint는 '양날의 검' 지금 데이터셋에는 `Broadcast`가 빠를 수 있어. 
> 근데 내년에 데이터가 10배 커지면? 강제로 걸어둔 `Broadcast` 힌트 때문에  **메모리 터지고(OOM)** 난리가 날 거야. 
> 그러니 **Hint는 정말 최후의 수단(Last Resort)** 으로만 쓰고, 웬만하면 최신 스파크(3.x)의 **AQE**에게 맡기는 게 정신 건강에 좋아!"
