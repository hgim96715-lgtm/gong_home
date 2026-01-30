---
aliases:
  - Broadcast
  - 브로드캐스트
  - BroadcastJoin
  - 셔플최적화
  - Spark_Broadcast
tags:
  - Spark
related:
  - "[[Spark_DataFrame_Basics]]"
  - "[[Spark_Functions_Library]]"
  - "[[Python_Classes_Objects]]"
  - "[[00_Apache_Spark_HomePage]]"
---
##  개념 한 줄 요약 

**"작은 데이터(참조표, 딕셔너리)를 드라이버가 포장해서 모든 워커(Executor)에게 미리 나눠주는 기술."**

* **목적:** 데이터를 매번 네트워크로 전송하거나, 무거운 **셔플(Shuffle)** 을 일으키는 것을 방지함.
* **비유:** 시험 칠 때 선생님(Driver)이 칠판에 '참조표'를 적어두는 것. (학생들은 고개만 들면 바로 알 수 있음)

---
##  Python 객체(Dict) 브로드캐스팅 & UDF 

실무에서 코드성 데이터(ID -> Name 변환)를 처리할 때 가장 많이 쓰는 **"Dictionary Lookup"** 패턴입니다.

### ① 데이터 준비 (Driver Side)

드라이버 메모리에 존재하는 일반적인 파이썬 딕셔너리입니다.

```python
meta = {
    "1100": "engineer",
    "2030": "developer",
    "3801": "painter",
    "3021": "chemistry teacher",
    "9382": "priest"
}
```

### ② 브로드캐스트 변수 생성 (The Sender) 

드라이버에 있는 데이터를 **포장(객체화)** 해서 클러스터 전체에 **배포**합니다.

```python
# [핵심] 딕셔너리를 Broadcast 객체로 포장해서 배포
occupation_dict = spark.sparkContext.broadcast(meta)
```

- **`sparkContext`:** 스파크 클러스터 관리자입니다. 얘가 배송 기사 역할을 합니다.
- **작동 원리:** 이 코드가 실행되는 순간, `meta` 데이터는 압축되어 모든 Executor 노드의 메모리로 전송됩니다. (네트워크 전송은 딱 1번만 발생)

### ③ 로직 구현과 `.value` 접근 (The Logic) 

워커 노드에서 실행될 함수를 정의합니다. 여기서 **`.value`** 가 등장합니다.

```python
def get_occupation_name(occupation_id: str) -> str:
    # 🚨 주의: occupation_dict는 딕셔너리가 아니라 '박스(Broadcast 객체)'입니다.
    # 반드시 .value를 붙여야 박스 안의 '내용물(dict)'을 꺼낼 수 있습니다.
    return occupation_dict.value.get(occupation_id, "Unknown")
```

- **왜 `.value`를 써야 하나?:** `occupation_dict`는 데이터 그 자체가 아니라, 데이터를 감싸고 있는 핸들(Handle)입니다. 
- 껍질을 벗겨야 딕셔너리 메서드(`.get`)를 쓸 수 있습니다.
- **특징:** 이 값은 **Read-Only(읽기 전용)** 입니다. 워커에서 수정할 수 없습니다.

> [!tip] **잠깐! `.value` 문법이 낯선가요?** 스파크 문법이 아니라 **파이썬 객체(Object)** 문법입니다. 
>헷갈린다면 **[[Python_Classes_Objects#③ `.value` 같은 건 어디서 온 거예요?|👉 value의 정체와 행방]]** 문서를 먼저 읽어보세요!

### ④ UDF 등록 (The Translator) 

파이썬 함수를 스파크 SQL 엔진이 이해할 수 있도록 변환합니다.

```python
import pyspark.sql.functions as F
from pyspark.sql.types import StringType

# 파이썬 함수 -> 스파크 SQL 함수(UDF)로 변환 (통역사 등록)
occupation_lookup_udf = F.udf(get_occupation_name, StringType())
```

- **`F.udf`가 뭔가요?:** User Defined Function(사용자 정의 함수). 
- 스파크(JVM 기반)는 파이썬 코드를 직접 실행할 수 없어서, **"이 컬럼 처리할 때는 파이썬한테 물어보고 와"** 라고 연결해주는 다리(Bridge) 역할을 합니다.
- **`StringType()`:** 반환 타입을 명시해주면 성능 최적화에 도움이 됩니다.

### ⑤ 적용 및 실행 (Execution)

```python
# 조인(Join) 없이, 맵핑(Map)만으로 데이터 채우기
df_result = interviewer_count.withColumn(
    "occupation_name", 
    occupation_lookup_udf(F.col("occupation_id"))
)

df_result.show()
```

---
## 실전 패턴 2: DataFrame 브로드캐스트 조인 

참조 데이터가 딕셔너리가 아니라 **"또 다른 데이터프레임(DataFrame)"** 일 때 씁니다.

### 상황

- `big_df`: 10억 건의 로그 데이터
- `small_df`: 100건의 사용자 정보 테이블

**코드 예제**

```python
import pyspark.sql.functions as F

# [핵심] 작은 테이블 쪽에 F.broadcast() 씌우기
# 스파크에게 "이거 작으니까 셔플하지 말고 그냥 다 복사해서 뿌려!"라고 강제 명령
joined_df = big_df.join(F.broadcast(small_df), "user_id")

joined_df.explain() 
# 실행계획(Physical Plan)에 'BroadcastHashJoin' 확인 필수!
```

**실행 계획 해석 (Evidence)**

`explain()` 로그에서 아래 키워드가 보이면 **성공**입니다!

```text
== Physical Plan ==
...
*(1) BroadcastHashJoin [user_id#0], [user_id#10], Inner, BuildRight
:- *(1) Project ... (Big Data)
+- BroadcastExchange HashedRelationBroadcastMode(List(input[0, string, true])), [id=#20] 
   +- LocalTableScan ... (Small Data)
```

- **`BroadcastExchange`**: "작은 데이터를 포장해서 전송 준비 중이야."
- **`HashedRelationBroadcastMode`**: "데이터를 해시맵 형태로 만들어서 뿌릴 거야." (조인 속도 최적화
- **결론**: 데이터를 모든 Executor로 복사했으므로 **셔플(Shuffle)이 발생하지 않음.** (네트워크 비용 절감)

>[!tip] **Spark UI에서도 확인 가능!**
>브라우저에서 `Spark UI` 
> `SQL / DataFrame` 탭을 클릭해보세요.
>  실행 그래프(DAG)에서 **`BroadcastExchange`** 라는 박스가 보인다면 제대로 적용된 것입니다.

---
## 언제 무엇을 써야 할까? (Comparison)

|**구분**|**패턴 1 (Variable + UDF)**|**패턴 2 (Join + F.broadcast)**|
|---|---|---|
|**참조 데이터**|Python `dict`, `list`, `set`|Spark `DataFrame`|
|**사용 방식**|`sc.broadcast(변수)`|`F.broadcast(df)`|
|**장점**|복잡한 로직(If/Else) 처리가 가능함|스파크 최적화 엔진(Catalyst) 활용으로 **가장 빠름**|
|**단점**|UDF 통신 오버헤드(Python↔JVM) 발생|조인 조건이 단순해야 함|


---
## 주의사항 (Warning) ⚠️

**"메모리는 무한하지 않다."**

브로드캐스트는 데이터를 **메모리에 통째로** 올리는 방식입니다.

- 참조 데이터가 **너무 크면 (보통 100MB 이상)** Executor 메모리가 터져서 죽어버립니다 (**OOM Error**).
- **Driver 메모리**도 조심해야 합니다. (처음에 Driver가 데이터를 모아서 뿌리기 때문)


---
## 6. [실험] 성능 차이 검증 (Benchmark) step13.ipynb 참고 

`explain()`(실행 계획) 상으로는 차이가 없어 보이지만, 실제 **Spark UI**와 **체감 성능**에서는 분명한 차이가 발생합니다.

### 📊 테스트 결과 (Small Data 기준)
| 구분 | 실행 시간 (Duration) | 체감 성능 (UX) | 원인 분석 |
| :--- | :--- | :--- | :--- |
| **No Broadcast** | **97 ms** | 결과 출력 시 **버벅임(Lag) 발생** 🐢 | 함수가 실행될 때마다 딕셔너리를 매번 포장(Serialization)해서 보내느라 오버헤드 발생. |
| **Broadcast** | **81 ms** (`-16ms`) | **빠르게 결과 출력** ⚡️ | 딕셔너리는 이미 가 있고, 함수는 '열쇠'만 들고 가볍게 실행됨. |

> **💡 Insight**
> 겨우 5개짜리 데이터에서도 이 정도 차이가 난다면, **실무 데이터(수만 건)** 에서는 **몇 분 vs 몇 시간**의 차이로 벌어질 수 있음!


> [!example] **📂 실험 소스 코드 (Jupyter Notebook)**
> 직접 돌려본 코드는 여기에 저장되어 있음.
>
> * **파일 경로:** `/Users/gong/gong_study_de/apache-spark/notebooks/step13.ipynb`
> * **주요 내용:**
>     * `BatchEvalPython` 실행 계획 비교 확인
>     * `Broadcast` 적용 전후 실행 시간 측정 (97ms vs 81ms)
>[📓 노트북 바로 열기](file:///Users/gong/gong_study_de/apache-spark/notebooks/step13.ipynb)
