---
aliases:
  - Catalyst Optimizer
  - Logical Plan
  - Physical Plan
  - AQE
tags:
  - Spark
related:
  - "[[Spark_Architecture]]"
  - "[[Spark_Transformations_vs_Actions]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_AQE_Deep_Dive]]"
  - "[[Spark_Caching_Strategies]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step17.ipynb
---
## 개념 한 줄 요약 

**"우리가 쓴 SQL/DataFrame 코드를 스파크가 이해할 수 있는 가장 효율적인 RDD 명령어로 바꿔주는 '번역기'이자 '최적화 엔진'."**

* **Logical Plan:** "무엇(What)"을 할 것인가? (추상적 계획)
* **Physical Plan:** "어떻게(How)" 할 것인가? (구체적 실행 전략)

---
## 전체 변환 과정 (The Lifecycle) 

스파크 SQL 엔진(Catalyst)은 다음 4단계를 거쳐 코드를 실행합니다

### ① Analysis (분석) - "문법 검사"

* **Unresolved Logical Plan:** 코드를 처음 파싱한 상태입니다. 테이블 이름이나 컬럼이 진짜 존재하는지 모릅니다
* **Catalog:** 메타데이터 저장소(데이터 사전)를 뒤져서 컬럼명과 타입을 확인합니다
* **Analyzed Logical Plan:** 확인이 끝나면 "검증된 계획"이 됩니다

### ② Logical Optimization (논리적 최적화) - "지능형 튜닝"

* **Optimized Logical Plan:** 비효율적인 로직을 똑똑하게 바꿉니다
* **주요 최적화 기법:** 
    * **Predicate Pushdown:** `Filter`를 데이터 읽는 곳(Source)까지 밀어 넣어서, 처음부터 필요한 데이터만 가져옵니다.
    * **Projection Pruning:** 안 쓰는 컬럼은 아예 읽지도 않습니다.
    * **Constant Folding:** `1 + 2` 같은 상수는 미리 `3`으로 계산해둡니다.

### ③ Physical Planning (물리적 계획) - "전략 수립"

* **Candidate Plans:** 논리적 계획을 바탕으로 "어떻게 실행할지" 여러 가지 전략(Join 방식 등)을 짭니다
* **Cost Model (CBO):** 각 전략의 비용(파일 크기, 셔플 비용 등)을 계산해서 **가장 저렴한(Best) 계획**을 선택합니다
* **Final Physical Plan:** 최종적으로 선택된 실행 계획입니다

### ④ Code Generation (코드 생성) - "실행"

* **Whole-Stage Codegen:** 최종 계획을 **Java Bytecode**로 변환하여 기계가 실행하게 만듭니다(RDD 변환)

---
## Logical Plan vs Physical Plan 비교 

| 구분      | Logical Plan (논리)                 | Physical Plan (물리)                            |
| :------ | :-------------------------------- | :-------------------------------------------- |
| **관점**  | **"무엇을(What)"** 할 것인가?            | **"어떻게(How)"** 실행할 것인가?                       |
| **내용**  | `Filter`, `Select`, `Join` (추상적)  | `FileScan`, `HashJoin`, `SortMergeJoin` (구체적) |
| **예시**  | "직원 테이블에서 나이가 30 이상인 사람 이름 찾아줘."  | "Parquet 파일 열고, 필터 걸어서 스캔하고, 셔플해서 조인해."       |
| **의존성** | 데이터 위치나 클러스터 상태 모름                | 데이터 분포와 클러스터 리소스를 고려함                         |

---
##  AQE (Adaptive Query Execution) 

Spark 3.0부터 도입된 게임 체인저입니다.
과거에는 계획을 한 번 짜면 끝까지 밀고 나갔지만, **AQE는 실행 도중에 계획을 바꿉니다.**

### 작동 원리 

1.  **초기 계획 수립:** 일단 예상치(Estimate)로 계획을 짭니다.
2.  **중간 점검 (Runtime):** 셔플(Map 단계)이 끝나면 **실제 데이터 통계(크기, 분포)** 를 확인합니다
3.  **계획 수정 (Re-planning):** 예상보다 데이터가 작거나 불균형하면 즉시 전략을 수정합니다

### 주요 기능 3가지

1.  **Dynamically Coalesce Shuffle Partitions:**
    * 데이터가 적으면 파티션을 합쳐서 **작은 파일 문제(Small File Problem)** 를 해결합니다
2.  **Dynamically Switch Join Strategies:**
    * 원래 `SortMergeJoin`을 하려 했는데, 막상 까보니 데이터가 작으면 **`BroadcastJoin`** 으로 바꿔버립니다 (속도 급상승)
3.  **Optimize Skew Joins:**
    * 특정 키에 데이터가 쏠려있으면(Skew), 알아서 쪼개서 병렬 처리합니다

> 더 자세한 내용은 [[Spark_AQE_Deep_Dive]] 참고 

---
## 5. 실전: `.explain(True)` 해부하기  > step17.ipynb

백문이 불여일견! 실제로 `df.explain(True)`를 찍었을 때 나오는 로그를 해석해봅시다.

- 예시 : `SELECT name FROM employees WHERE age > 30`

### [Step 1] Parsed Logical Plan (파싱된 논리적 계획)

"문법(Syntax)은 통과했지만, 아직 대상의 실체는 모르는 상태"

```text
== Parsed Logical Plan ==
'Project ['name]
+- 'Filter ('age > 30)
   +- 'UnresolvedRelation [employees], [], false
```

- **`'Project ['name]`**: "사용자가 `name` 컬럼을 조회(SELECT)하겠다고 했음."
- **`'Filter ('age > 30)`**: "`age`가 30보다 크다(WHERE)는 조건을 걸었음."
- **`'UnresolvedRelation [employees]`**: **[핵심]** "근데 `employees`가 뭐야? 테이블이야? 뷰야? 파일이야?"
	- 아직 **카탈로그(Catalog)** 를 확인하지 않아서 실체를 모르는 **'미해결(Unresolved)'** 상태입니다.
	- `age`가 숫자인지 문자인지도 모릅니다. 식별자(Identifier)만 확인한 단계입니다.

### [Step 2] Analyzed Logical Plan (분석된 계획)

"문법은 맞는지, 컬럼은 진짜 있는지 확인하는 단계"

```text
== Analyzed Logical Plan ==
name: string
Project [name#1]
+- Filter (age#2L > cast(30 as bigint))
   +- SubqueryAlias employees
      +- View (`employees`, [id#0L,name#1,age#2L])
	       +- LogicalRDD [id#0L, name#1, age#2L], false
```

- **`Project [name#1]`**: "`name` 컬럼만 뽑아내겠다(Select)." (`#1`은 스파크 내부 ID)
- Filter (age#2L > cast(30 as bigint)) : 스파크가 `age` 컬럼 타입(`Long`, #2L)에 맞춰서 `30`을 `bigint`로 **자동 형변환(Cast)** 했습니다
- **`SubqueryAlias employees`**: "이 데이터의 이름은 `employees`다."
- **`View`**: "이건 원본 테이블이 아니라, 사용자가 만든 **임시 뷰(Temp View)**다."
- **`LogicalRDD`**: "데이터 소스가 파일(Parquet/CSV)이 아니라, **이미 메모리에 있는 DataFrame/RDD**다."
> 파케이 파일을 읽었으면 LogicalRDD가 아니라 **`Relation ... parquet`** 라고 뜬다.!

### [Step 3] Optimized Logical Plan (최적화된 계획)

"비효율적인 껍데기를 벗기고, 논리적인 오류를 사전에 차단하는 단계"

```text
== Optimized Logical Plan ==
Project [name#1]
+- Filter (isnotnull(age#2L) AND (age#2L > 30))
   +- LogicalRDD [id#0L, name#1, age#2L], false
```

- **`isnotnull(age#2L)` 추가 (중요!)**:
	-  사용자는 `age > 30`만 적었지만, 스파크는 **"NULL은 어차피 30보다 클 수 없다"**는 것을 알고 있습니다.
	- 그래서 **`NULL` 체크 조건을 자동으로(공짜로) 추가**했습니다. 이렇게 하면 NULL 데이터를 미리 걸러내어 불필요한 비교 연산을 줄일 수 있습니다.
- **`SubqueryAlias`, `View` 삭제**:
	- Analyzed 단계에 있던 `SubqueryAlias employees`나 `View` 같은 껍데기들은 실행에 필요 없으므로 과감히 **제거(Pruning)** 했습니다.
	- 오직 데이터 처리에 필요한 알맹이(`Project`, `Filter`, `LogicalRDD`)만 남겼습니다.

### [Step 4] Physical Plan (물리적 실행 계획)

"실제로 CPU가 수행할 구체적인 명령어 세트"

```text
== Physical Plan ==
*(1) Project [name#1]
+- *(1) Filter (isnotnull(age#2L) AND (age#2L > 30))
   +- *(1) Scan ExistingRDD[id#0L,name#1,age#2L]
```

- **`Scan ExistingRDD`**:
	- 파일 스캔(`FileScan`)이 아니라, **"이미 메모리에 존재하는 RDD(DataFrame)를 훑겠다"** 는 뜻입니다.
	- 만약 파케이 파일을 읽었다면 `Scan parquet ... PushedFilters[...]`가 떴을 것입니다.

- `*(1)` (Whole-Stage Codegen)
	- 이 별표(`*`)가 가장 중요합니다.
	- "이 단계들(Scan, Filter, Project)을 따로따로 실행하지 않고, **하나의 자바 함수(Bytecode)로 퉁쳐서 최적화했다**"는 뜻입니다.
	- 이를 **Whole-Stage Code Generation**이라고 하며, 함수 호출 오버헤드를 없애 속도를 비약적으로 높여줍니다.

![[스크린샷 2026-02-02 오후 2.28.47.png|200]]

> ▲ Spark UI의 SQL 탭에서 본 모습. 파란색 박스가 바로 Whole-Stage Codegen으로 합쳐진 구간입니다!
> **`Scan ExistingRDD`**: 이미 있는 RDD를 훑습니다.
> **`WholeStageCodegen` (파란 박스)**: Scan, Filter, Project가 하나의 박스 안에 있는것은 따로따로 실행되는 게 아니라, **하나의 함수로 합쳐져서 실행**된다는 증거입니다.