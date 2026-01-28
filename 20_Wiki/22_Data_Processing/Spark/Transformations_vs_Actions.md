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
##  개념 한 줄 요약

**"스파크는 '계획(Transformation)'만 세우다가, '결과 내놔(Action)'라고 소리쳐야 비로소 움직인다."**

* **Transformation (대기조):** 데이터를 변형하는 **설계도**만 그립니다. (계획만 세움 / **Lazy**)
* **Action (활동조):** 실제 계산을 수행하고 **결과**를 가져옵니다. (실행 / **Eager**)

---
## Transformation (변환 / 계획조) 

기존 RDD를 변경하여 **새로운 RDD**를 만들어냅니다.
하지만 실제로 계산하지는 않고, **족보(Lineage)** 에 "나중에 이거 해야 함"이라고 적어두기만 합니다. **(Lazy Evaluation)**

### 중요: 셔플(Shuffle) 유무로 나뉩니다!

#### ① Narrow Transformation (좁은 변환) - "안전함" 

데이터가 네트워크를 타고 이동할 필요 없이, 내 자리(Partition)에서 뚝딱 처리가 가능한 애들입니다.

- **`parallelize()`**: **[생성]** 리스트 데이터를 RDD로 만듭니다. (테스트할 때 필수!) , 데이터 주입 
* **`map()`**: 모양 바꾸기(1개를 1개로 변환)
* **`filter()`**: 필터링(조건에 맞는 것만 남기기)
* **`flatMap()`**: 납작하게 펴기(1개를 여러 개로 뻥튀기)
* **`mapValues()`**: 값만 바꾸기(Key 유지, 파티션 유지)
* **`sample()`**: **[추출]** 랜덤하게 일부만 뽑아서 **새로운 RDD**를 만듭니다.
	* `withReplacement`: 복원 추출 여부 (True/False)
	* fraction`: 뽑을 비율 (예: 0.1 = 10%)
	* *주의: `takeSample`(Action)과 다릅니다! 이건 결과를 RDD로 줍니다.*
- **`glom()`**: **[디버깅]** 각 파티션에 있는 데이터를 **하나의 리스트로 묶어줍니다.**
	- 용도: "내 데이터가 파티션별로 예쁘게 잘 나뉘어 있나(Skew 확인)?" 궁금할 때 씁니다.
	- 예: `[1, 2, 3, 4]` -> `[[1, 2], [3, 4]]` (파티션 경계가 보임)

#### ② Wide Transformation (넓은 변환) - "주의 요망" 

데이터가 노드 간에 뒤섞여야(Shuffle) 하는 애들입니다. 
**Narrow보다 무조건 느리지만**, 집계를 하려면 어쩔 수 없이 써야 합니다.

> 참고 : **Narrow (`map`)**: 내 방에서 나 혼자 처리함. **(비용 0, 초고속)**

* **`reduceByKey()`**:**[추천]** 키별로 모아서 합치기
	* *왜 추천인가요?* 셔플이 발생하지만, **데이터를 미리 줄여서 보내므로(Map-side Combine)** Wide 중에선 제일 빠릅니다.
* **`groupByKey()`**: **[비추]** 키별로 뭉치기
	* * *왜 비추인가요?* 데이터를 쌩으로 다 보내서 네트워크가 터질 수 있습니다.
* **`distinct()`**: 중복 제거 (전체 데이터를 다 뒤져야 함 -> 셔플 발생)
* **`sortByKey()`**: 정렬하기 (데이터 순서를 맞추려면 셔플 필수)
* **`repartition()`**: 파티션 개수 늘리기/줄이기 (강제 셔플)

---
##  Action (행동 / 실행조) 

족보(DAG)를 보고 드디어 **실제로 계산(Compute)** 을 수행합니다.
결과값으로 **RDD가 아닌 것(리스트, 숫자, 파일 등)** 을 내놓습니다.

### ① 데이터를 눈으로 볼 때 (View)

* **`collect()`**: 🚨 **[주의]** 모든 데이터를 Driver 메모리로 가져옵니다. (데이터 크면 서버 터짐 OOM)
* **`take(n)`**: 앞에서부터 n개만 안전하게 가져옵니다. (빠름, but 정렬 안 되어 있으면 랜덤이나 마찬가지)
* **`first()`**: 맨 앞에 있는 딱 1개만 가져옵니다. (`take(1)`과 같음)
* **`top(n)`**: (내림차순 정렬해서) 가장 큰 n개를 가져옵니다.
* **`takeSample(withReplacement, num, seed)`**: **랜덤으로 n개를 뽑아옵니다.*
	* `withReplacement`: 복원 추출 여부 (`True`면 뽑은 거 또 뽑을 수 있음) 
	* `num`: 뽑을 개수 
	* `seed`: 랜덤 시드 (이걸 넣어야 항상 똑같은 랜덤 결과가 나옴)

### ② 계산할 때 (Math)

* **`count()`**: 데이터 개수가 몇 개인지 셉니다.
* **`countByValue()`**: 각 값이 몇 번 나왔는지 세어서 **딕셔너리**로 줍니다.
* **`reduce(func)`**: 🚨 **[주의]** `reduceByKey`랑 다름! 모든 데이터를 하나로 합쳐서 **단 하나의 값(숫자)** 을 냅니다. (예: 전체 총합)

### ③ 저장할 때 (Save)

* **`saveAsTextFile(path)`**: 텍스트 파일로 저장합니다.
* **`saveAsObjectFile(path)`**: 파이썬 객체 그대로 저장합니다.


###  [면접 질문] `sample()` vs `takeSample()` 차이가 뭔가요?

| 구분      | **sample()**                | **takeSample()**             |
| :------ | :-------------------------- | :--------------------------- |
| **소속**  | **Transformation** (변환)     | **Action** (행동)              |
| **반환값** | **새로운 RDD** (작아진 RDD)       | **파이썬 리스트** (Driver로 가져옴)    |
| **용도**  | "데이터가 너무 많으니 10%만 남기고 작업하자" | "데이터 상태 좀 보게 랜덤으로 10개만 가져와봐" |
| **주의**  | 계산 안 하고 기다림 (Lazy)          | **즉시 실행됨 (Eager)**           |

---
##  왜 굳이 '게으른 연산(Lazy)'을 하나요? 🤔

**"최적의 경로(지름길)를 찾기 위해서"** 입니다.

1.  사용자: "1TB 데이터 읽어." (`Transformation`)
2.  사용자: "거기서 `map` 하고..." (`Transformation`)
3.  사용자: "음, 다시 생각해보니 `filter`로 99%는 버려." (`Transformation`)
4.  사용자: "이제 `count` 해봐!" (**Action!**)

> **스파크의 생각:**
> "아하, 어차피 99%는 버릴(`filter`) 거잖아?
> 그럼 처음부터 1TB를 다 `map` 하지 말고, **`filter`를 먼저 적용해서 데이터를 줄인 다음에** `map`을 하자!"
> (이것이 스파크의 **Catalyst Optimizer**가 하는 일입니다.)

---
###  (실수 방지)

1.  **`print()`의 함정:**
    * Transformation 안에 `print()`를 넣어도 콘솔에 아무것도 안 찍혀. 아직 실행 안 했으니까!
    * 반드시 끝에 `collect()`나 `count()` 같은 Action을 붙여야 비로소 출력이 돼.
2.  **`collect()` 금지령:**
    * 실무 데이터는 몇 억 건이야. 이걸 `collect()` 하는 순간 네 컴퓨터(Driver)는 뻗어버려.
    * 데이터 확인하고 싶으면 무조건 **`take(5)`** 나 **`show(5)`** 를 쓰는 습관을 들여!

