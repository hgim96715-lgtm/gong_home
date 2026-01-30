---
aliases:
  - Graph Analysis
  - Social Network
  - collect_set
  - concat_ws
  - size
  - Array Functions
  - 그래프분석
tags:
  - Spark
related:
  - "[[Spark_DataFrame_Basics]]"
  - "[[Spark_Functions_Library]]"
linked: file:///Users/gong/gong_study_de/apache-spark/notebooks/step14.ipynb
---
## 개념 한 줄 요약 

**"흩어져 있는 1:1 관계 데이터(로그)를 1:N 형태(리스트)로 묶어서, '연결의 구조'와 '중심(Hub)'을 파악하는 과정."**

**Edge List (엣지 리스트/로그):**
- `출발(Source) -> 도착(Target)` 형태로, 관계가 한 줄에 하나씩 기록된 원본 데이터. 
- (예: 결제 로그, 접속 로그)

**Adjacency List (인접 리스트/배열):**
- `출발(Source) -> [도착1, 도착2, 도착3...]` 처럼, 하나의 주체에 연결된 모든 대상을 **배열(List)** 로 묶은 형태.

> **💡 왜 범용적인가요?** 
>  **커머스:** `User A -> [상품1, 상품2]` (장바구니 분석) 
>   **보안:** `IP주소 -> [접속한 서버1, 접속한 서버2]` (해킹 시도 탐지) 
>   **SNS:** `User A -> [친구1, 친구2]` (인맥 분석) > 데이터의 **모양**은 다 똑같기 때문입니다!


---
##  핵심 함수 3대장 

이 분석을 가능하게 하는 스파크 함수들입니다. (정말 자주 쓰입니다!)

| 함수                        | 설명                                          | 비유                   |
| :------------------------ | :------------------------------------------ | :------------------- |
| **`collect_set(col)`**    | 그룹별로 데이터를 긁어모아 **중복 없는 배열(Array)**로 만듦.     | 드래곤볼 모으기 (중복 X) 🟠   |
| **`concat_ws(sep, col)`** | 배열을 특정 구분자(`,`)로 이어 붙여 **문자열(String)**로 만듦. | 구슬을 실로 꿰어 목걸이 만들기 📿 |
| **`size(col)`**           | 배열이나 맵의 **크기(개수)**를 셈.                      | 목걸이에 구슬 몇 개인지 세기 🔢  |

---
##  코드 해부 (Code Analysis) 

### Step 1: 대상별 관계 리스트 만들기 (Aggregation)

분석하고 싶은 **주체(Key)** 를 기준으로, 연관된 **대상(Value)** 들을 싹 긁어모읍니다.

- 문법 : `{python}collect_set(col)` : 그룹별로 데이터를 긁어모아 **중복 없는 배열(Array)** 로 만듦.

```python
# '기준 컬럼(Key)'으로 그룹핑하고 -> '연관 컬럼(Target)'을 리스트로 묶음
# `hero1`을 기준으로 `hero2`들을 싹 긁어모읍니다.
data = df.groupBy("hero1")\
    .agg(
        # hero2를 모아서 중복 제거(set) 후 'connection' 컬럼에 넣음
        f.collect_set("hero2").alias("connection")
    )\
    .withColumnRenamed("hero1", "hero")
```

- **결과:** `주체(Key) | [대상1, 대상2, 대상3...]` (List/Array 타입)
- **의미:** "이 주체(Key)가 **누구와(What)** 연결되어 있는가?"를 한 줄로 요약함.

### Step 2: 하나의 파일로 모아서 저장 (Serialization & Export)

리스트(`[]`) 타입은 CSV로 저장이 안 되기 때문에 문자열로 바꾸고,
**흩어져 있는 데이터 조각을 하나로 합쳐서** 파일로 내보냅니다.

- 문법 : `{python}concat_ws(sep, col)` : 배열을 특정 구분자(`,`)로 이어 붙여 **문자열(String)** 로 만듦.

```python
# 1. 변환: ["Hulk", "Thor"] -> "Hulk,Thor" (리스트를 문자열로)
data = data.withColumn("connection", f.concat_ws(",", f.col("connection")))

# 2. 저장: [중요] 흩어진 파티션을 1개로 합쳐서(coalesce) CSV로 저장
data.coalesce(1).write.option("header", True).csv("output")
```

>[!info] **잠깐! 스파크의 저장 방식이 궁금하다면?**
>`coalesce` 없이 그냥 저장하면 폴더와 여러 개의 파일이 생깁니다. 
>왜 그런지 궁금하다면 **[[Spark_Data_IO|👉 데이터 저장의 비밀(Write & Folder)]]** 문서를 참조하세요!

### Step 3: 데이터 복원 및 연결 규모 측정 (Deserialization & Analysis)

문자열(String)로 저장된 데이터를 다시 **배열(List)** 로 복원하고,
그 **크기(Size)** 를 계산하여 **"누가 가장 영향력이 큰지(Hub)"** 순위를 매깁니다.

```python
df = df.withColumn(
        "connection_size",
        # 1. split: "A,B,C" -> ["A", "B", "C"] (문자열을 다시 리스트로 복원)
        # 2. size: 리스트의 길이 계산 -> 3개 (연결된 대상의 수/트래픽/구매량 등)
        f.size(f.split(f.col("connection"), ","))
    )\
df.orderBy(f.desc("connection_size")).show() # 규모가 큰 순서대로(내림차순) 정렬
```

---
## 왜 이렇게 하나요? (Why)

**"관계(Relationship)가 중요한 데이터니까!"**

일반적인 통계(`avg`, `sum`)로는 **"누가 중심인가?"** 를 알 수 없습니다.

- **추천 시스템:** "너랑 친구인 애들이 이거 샀어."
- **이상 탐지:** "평소랑 다르게 갑자기 많은 사람에게 돈을 보냈네?" 이런 분석을 할 때 **`collect_set`으로 묶어서 패턴을 보는 것**이 기본입니다.

---

##  [심화] 굳이 저장해야 하나요? (Optimization) 🚀

**"아니요! 메모리에서 바로 계산하는 게 훨씬 빠릅니다."**

위의 예제는 **[데이터 저장]** 과 **[불러오기]** 과정을 보여주기 위한 실습용 코드입니다.
실무에서 단순히 결과값만 필요하다면, **저장(I/O) 없이 한 번에** 끝내는 것이 정답입니다.

### ✅ 최적화된 코드 (In-Memory Processing)

중간에 파일로 내렸다가 다시 올리는 과정(Disk I/O)을 없애고,
**`collect_set`으로 모으자마자 바로 `size`를 잽니다.**

```python
# 저장했다 불러오는 과정 싹 생략! ✂️

real_result = df.groupBy("hero1")\
    .agg(
        # 1. collect_set: 모으고
        # 2. size: 바로 크기 재기 (함수 안에 함수 넣기)
        f.size(f.collect_set("hero2")).alias("connection_size")
    )\
    .withColumnRenamed("hero1", "hero")\
    .orderBy(f.desc("connection_size"))

real_result.show()
```

**성능 비교**

| **구분** | **원래 코드 (Disk I/O)**                                 | **최적화 코드 (In-Memory)** |
| ------ | ---------------------------------------------------- | ---------------------- |
| **과정** | 배열 -> 문자열 -> **디스크 저장** -> **읽기** -> 문자열 -> 배열 -> 숫자 | 배열 -> 숫자               |
| **비용** | 비쌈 (쓰고 읽는 시간 + 변환 시간)                                | **매우 저렴 (0초 컷)**       |
| **용도** | 다른 팀에게 데이터를 파일로 전달할 때                                | 내가 바로 분석 결과만 보고 싶을 때   |


---
