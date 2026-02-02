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
  - RDD
related:
  - "[[RDD_Concept]]"
  - "[[Python_Lambda_Map]]"
  - "[[Python_Looping_Helpers]]"
  - "[[Python_Random_Seed]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Iterating_Data]]"
  - "[[Python_Control_Flow]]"
---
## 개념 한 줄 요약 

**"스파크는 '계획(Transformation)'만 세우다가, '결과 내놔(Action)'라고 소리쳐야 비로소 움직인다."**

* **Transformation (계획/대기조):** 데이터를 변형하는 설계도만 그립니다. (Lazy Evaluation) 
* **Action (실행/활동조):** 실제 계산을 수행하고 결과를 Driver로 가져오거나 저장합니다. (Eager Execution) 
* **핵심:** 계획 단계에서 **셔플(Shuffle)** 발생 여부(Narrow vs Wide)가 성능을 좌우합니다.

---
##  Transformation (변환 / 계획조) 

기존 RDD/DataFrame을 변경하여 **새로운 데이터**를 만들어냅니다.
하지만 실제로 계산하지는 않고, **족보(Lineage)** 에 "나중에 이거 해야 함"이라고 적어두기만 합니다.

### 중요: 셔플(Shuffle) 유무에 따른 분류 

성능 튜닝의 핵심입니다.데이터가 이동하느냐 안 하느냐로 나뉩니다.

#### ① Narrow Transformation (좁은 변환) - "빠르고 안전함" 

부모 파티션의 데이터가 **자식 파티션 하나**에만 그대로 전달됩니다

* **특징:**
    * **No Shuffle:** 데이터가 네트워크를 타고 이동할 필요 없이, 내 자리(Node)에서 뚝딱 처리합니다. 
    * **고속 처리 & 빠른 복구:** 에러 발생 시 해당 파티션만 다시 계산하면 됩니다.
* **주요 함수 (RDD/DataFrame):**
    * **`map()`**: 모양 바꾸기 (1개를 1개로 변환)
    * **`filter()`**: 필터링 (조건에 맞는 것만 남기기)
    * **`flatMap()`**: 납작하게 펴기 (1개를 여러 개로 뻥튀기)
    * **`mapValues()`**: (PairRDD) Key는 유지하고 Value만 바꿈.
    * **`union()`**: 두 데이터를 합치기.
    * **`select()`, `withColumn()`, `drop()`**: (DataFrame) 컬럼 조작.
    * **`sample()`**: 랜덤하게 일부만 추출하여 **새로운 RDD** 생성.
    * **`glom()`**: 각 파티션의 데이터를 하나의 리스트로 묶음 (데이터 분포 디버깅용).

#### ② Wide Transformation (넓은 변환) - "느리고 비쌈 (Shuffle)" 🐢

부모 파티션의 데이터가 **여러 자식 파티션으로 쪼개져서** 들어갑니다

* **특징:**
    * **Shuffle 발생:** 데이터를 키(Key) 기준으로 모으기 위해 **네트워크를 타고 다른 노드로 이동**합니다.
    * **비용 발생:** 디스크 I/O와 네트워크 트래픽이 발생하여 속도가 느립니다.
    * **느린 복구:** 파티션 하나가 깨지면, 연관된 모든 부모 파티션을 다시 계산해야 할 수 있습니다.
* **주요 함수 (RDD/DataFrame):**
    * **`reduceByKey()`** **[추천]**: 키별로 합치기 (Map-side Combine으로 데이터 양을 줄여서 보냄).
    * **`groupByKey()`** **[비추]**: 키별로 뭉치기 (데이터를 쌩으로 다 보내서 네트워크 부하 큼).
    * **`sortByKey()` / `orderBy`**: 정렬하기 (전체 순서를 맞추기 위해 이동 필수).
    * **`distinct()`**: 중복 제거 (전체 데이터를 뒤져야 함).
    * **`join()`**: 두 데이터를 합치기 (Join).
    * **`repartition()`**: 파티션 개수 재조정 (강제 셔플).

---
## Action (행동 / 실행조) 

족보(Lineage)를 보고 드디어 **실제로 계산(Compute)** 을 수행합니다
결과값으로 RDD가 아닌 **리스트, 숫자, 파일 등**을 내놓습니다.

### ① 데이터를 눈으로 볼 때 (View)

* **`collect()`**: 🚨 **[주의]** 모든 데이터를 Driver 메모리로 가져옵니다. (OOM 위험)
* **`take(n)`**: 앞에서부터 n개만 안전하게 가져옵니다.
* **`show()`**: (DataFrame) 데이터를 표 형태로 예쁘게 출력합니다.
* **`first()`**: 맨 앞의 1개만 가져옵니다.
* **`takeSample(withReplacement, num, seed)`**: 랜덤으로 n개를 뽑아 **리스트**로 가져옵니다.

### ② 계산할 때 (Math)

* **`count()`**: 데이터 개수를 셉니다.
* **`countByValue()`**: 각 값이 몇 번 나왔는지 셉니다.
* **`reduce(func)`**: 모든 데이터를 하나로 합쳐서 **단 하나의 값**을 냅니다.

### ③ 저장할 때 (Save)

* **`saveAsTextFile(path)`**: 텍스트 파일로 저장.
* **`saveAsObjectFile(path)`**: 파이썬 객체로 저장.
* **`write`**: (DataFrame) Parquet, CSV 등으로 저장.

---
## 비교 분석 및 요약표 (Cheat Sheet) 

### ① API 분류표 (Shuffle 유무)

| 구분 | Narrow (No Shuffle) | Wide (Shuffle) 🚨 | Action (Execute) 🔥 |
| :--- | :--- | :--- | :--- |
| **RDD** | `map`, `filter`, `flatMap`<br>`union`, `sample` | `reduceByKey`, `groupByKey`<br>`join`, `distinct`, `repartition` | `collect`, `count`<br>`take`, `saveAsTextFile` |
| **DataFrame** | `select`, `filter/where`<br>`withColumn`, `drop` | `groupBy`, `join`, `agg`<br>`orderBy`, `distinct` | `show`, `count`<br>`collect`, `write` |

### ② [면접 단골] `sample()` vs `takeSample()`

| 구분 | **sample()** | **takeSample()** |
| :--- | :--- | :--- |
| **소속** | **Transformation** (변환) | **Action** (행동) |
| **반환값** | **새로운 RDD** (작아진 데이터셋) | **파이썬 리스트** (Driver로 가져옴) |
| **용도** | "데이터가 너무 많으니 10%만 남기고 작업하자" | "데이터 상태 좀 보게 10개만 가져와봐" |
| **특징** | 계산 안 하고 기다림 (Lazy) | **즉시 실행됨 (Eager)** |

---
## 왜 굳이 '게으른 연산(Lazy)'을 하나요? 

**"최적의 경로(지름길)를 찾기 위해서"** 입니다. (Catalyst Optimizer)

1.  사용자: "1TB 데이터 읽어." (`Transformation`)
2.  사용자: "거기서 `map` 하고..." (`Transformation`)
3.  사용자: "음, `filter`로 99%는 버려." (`Transformation`)
4.  사용자: "이제 `count` 해봐!" (**Action!**)

> **스파크의 생각:**
> "아하, 어차피 99%는 버릴(`filter`) 거잖아?
> 그럼 1TB를 다 `map` 하지 말고, **`filter`를 먼저 적용해서 데이터를 줄인 다음에** `map`을 하자!"

---
###  실수 방지 Tip

1.  **`print()`의 함정:**
    * Transformation 안에 `print()`를 넣어도 콘솔에 아무것도 안 찍힙니다. 아직 실행 안 했으니까요!
    * 반드시 `collect()`나 `count()` 같은 Action을 붙여야 비로소 출력됩니다.
2.  **`collect()` 금지령:**
    * 실무 데이터는 몇 억 건입니다. 이걸 `collect()` 하는 순간 컴퓨터(Driver)는 뻗어버립니다.
    * 데이터 확인하고 싶으면 무조건 **`take(5)`** 나 **`show(5)`** 를 쓰는 습관을 들이세요!
3.  **Narrow vs Wide 차이 (면접용):**
    * "**셔플(Shuffle)** 발생 유무입니다. Wide는 네트워크를 타기 때문에 비용이 비싸서 최적화가 필요합니다."