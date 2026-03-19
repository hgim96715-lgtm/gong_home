---
aliases:
  - Transformation
  - Action
  - Lazy_Evaluation
  - parallelize
  - takeSample
  - 지연연산
tags:
  - Spark
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Session_Context]]"
  - "[[Spark_Architecture]]"
---
# Spark_Transformations_vs_Actions

## 한 줄 요약

```
Transformation  계획만 세움 (Lazy — 실행 안 함)
Action          실제로 실행 (Eager — 즉시 계산)

Action 을 호출해야 비로소 Transformation 이 실행됨
```

---

---

# ① Transformation — 계획만 세우기 (Lazy)

```
기존 DataFrame / RDD 를 변형해서 새로운 데이터 설계도를 그림
실제 계산은 안 하고 족보(Lineage) 에 기록만 해둠
Action 이 호출될 때 한꺼번에 최적화해서 실행
```

## Narrow Transformation — Shuffle 없음 (빠름)

```
데이터가 네트워크를 타고 이동하지 않음
각 노드에서 자기 파티션 데이터만 처리
에러 시 해당 파티션만 다시 계산 → 빠른 복구
```

|함수|설명|
|---|---|
|`filter()` / `where()`|조건에 맞는 행만 남기기|
|`select()`|컬럼 선택|
|`withColumn()`|컬럼 추가 / 변환|
|`drop()`|컬럼 제거|
|`map()`|1개 → 1개 변환 (RDD)|
|`flatMap()`|1개 → 여러 개 변환 (RDD)|
|`union()`|두 데이터 합치기|

## Wide Transformation — Shuffle 발생 (느림) ⚠️

```
데이터가 키 기준으로 재배치되어야 해서
네트워크를 타고 다른 노드로 이동 (Shuffle)
디스크 I/O + 네트워크 트래픽 발생 → 비쌈
```

|함수|설명|주의|
|---|---|---|
|`groupBy()` / `groupByKey()`|키별로 그룹핑|데이터 쌩으로 이동 → 느림|
|`reduceByKey()`|키별로 합치기 (RDD)|이동 전 미리 합침 → 빠름 ✅|
|`join()`|두 데이터 조인|Shuffle 불가피|
|`orderBy()` / `sortByKey()`|정렬|전체 순서 맞추려면 이동 필수|
|`distinct()`|중복 제거|전체 데이터 비교 필요|
|`repartition()`|파티션 재조정|강제 Shuffle|

```
groupBy vs reduceByKey:
  groupByKey()  데이터 전부 이동 후 합침  → 느림 ❌
  reduceByKey() 각 노드에서 먼저 합친 후 이동 → 빠름 ✅
```

---

---

# ② Action — 실제 실행 (Eager)

```
족보를 보고 드디어 계산 수행
결과값으로 DataFrame 이 아닌 리스트 / 숫자 / 파일 반환
Action 이 호출된 횟수만큼 재계산 발생
```

## 데이터 확인

|함수|설명|주의|
|---|---|---|
|`show(n)`|표 형태로 n행 출력|DataFrame 전용|
|`take(n)`|앞에서 n개 리스트로 반환|안전 ✅|
|`first()`|첫 번째 행 반환||
|`collect()`|전체 데이터 Driver 로 가져옴|OOM 위험 ⚠️|
|`takeSample(False, n)`|랜덤 n개 리스트 반환||

## 계산

|함수|설명|
|---|---|
|`count()`|전체 행 수 (전수 조사)|
|`isEmpty()`|비어있는지 확인 (take(1) 방식 — 빠름)|
|`reduce(func)`|모든 데이터를 하나의 값으로 합침|
|`countByValue()`|각 값 등장 횟수|

## 저장

|함수|설명|
|---|---|
|`write.parquet(path)`|Parquet 파일 저장|
|`write.csv(path)`|CSV 파일 저장|
|`saveAsTextFile(path)`|텍스트 저장 (RDD)|

---

---

# ③ isEmpty() vs count() — 언제 뭘 쓸까? ⭐️

## 일반적인 경우 — isEmpty() 압도적으로 빠름

```
count() == 0:
  전체 데이터를 끝까지 다 세고 나서 "0개네" 판단
  수십만 건도 전부 카운팅 → 시간/메모리 낭비

isEmpty():
  내부적으로 take(1) 방식으로 동작
  데이터 1개만 꺼내보고 있으면 즉시 "안 비었어(False)" 반환
  수백 배 빠름
```

```python
# ✅ 단순히 비었는지만 확인할 때
if batch_df.isEmpty():
    return
```

## 카운트도 함께 필요한 경우 — count() 한 번만

```
문제:
  isEmpty() 로 확인하고
  로그 출력하려고 count() 또 호출하면
  → Action 이 2번 실행 → 카프카를 두 번 훑음

해결:
  count() 를 변수에 저장해서 재사용
  → Action 1번으로 isEmpty + count 동시 해결
```

```python
# ❌ Action 2번 발생
def write_to_postgres(batch_df, batch_id):
    if batch_df.isEmpty():         # Action 1번
        return
    batch_df.write...save()
    print(f"{batch_df.count()}건") # Action 2번 → 카프카 다시 훑음

# ✅ Action 1번으로 해결
def write_to_postgres(batch_df, batch_id):
    batch_count = batch_df.count()  # Action 1번 — 변수에 저장
    if batch_count == 0:
        return
    batch_df.write...save()
    print(f"[배치 {batch_id}] {batch_count}건 → er_realtime 적재")
```

```
정리:
  카운트 로그 안 남긴다  → isEmpty() 사용
  카운트 로그 필요하다   → count() 변수에 저장해서 재사용
  둘 다 혼용            → Action 2번 → 비효율
```

---

---

# ④ Lazy Evaluation — 왜 게으르게 하나?

```
목적: 최적화된 실행 계획(Catalyst Optimizer) 을 자동으로 만들기 위해

예시:
  1. spark.read.csv("1TB.csv")   ← Transformation (안 읽음)
  2. .map(...)                   ← Transformation (안 함)
  3. .filter(hvec > 0)           ← Transformation (안 함)
  4. .count()                    ← Action! 이제 실행

Spark 의 생각:
  "어차피 filter 로 99% 버릴 거잖아
   그럼 map 먼저 하지 말고
   filter 먼저 해서 데이터 줄인 다음 map 하자"
  → 실행 순서를 최적화해서 계산
```

---

---

# ⑤ 자주 하는 실수

```python
# ❌ Transformation 안에 print() 써도 안 나옴
rdd.map(lambda x: print(x))  # Action 호출 전이라 실행 안 됨

# ✅ Action 붙여야 출력됨
rdd.map(lambda x: x).take(5)
```

```python
# ❌ collect() 금지 — 수억 건 데이터 → Driver OOM
result = df.collect()

# ✅ 소량만 확인
result = df.take(5)
df.show(5)
```

---

---

# 한눈에 정리

|구분|Narrow Transformation|Wide Transformation|Action|
|---|---|---|---|
|Shuffle|❌ 없음|✅ 있음|-|
|속도|빠름|느림|즉시 실행|
|예시|filter / select / map|groupBy / join / distinct|count / show / write|

```
sample()    vs  takeSample():
  sample()      Transformation → 새로운 RDD 반환 (Lazy)
  takeSample()  Action         → 파이썬 리스트 반환 (즉시 실행)
```