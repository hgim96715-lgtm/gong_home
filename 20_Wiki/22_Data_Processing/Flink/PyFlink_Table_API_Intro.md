---
aliases:
  - PyFlink
  - API
  - TABLE
  - SQL
tags:
  - PyFlink
related:
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_Table_Expressions]]"
  - "[[PyFlink_SQL_Integration]]"
  - "[[PyFlink_SQL_Windows]]"
---
## **한 줄 요약:** 

**"Flink의 DataFrame."** (복잡한 자바/파이썬 객체 대신, **테이블과 컬럼(SQL)** 관점으로 데이터를 다루는 고수준 API)

---
## DataStream API vs Table API

| **특징**  | **DataStream API (기존)**                               | **Table API & SQL (신규)**                                  |
| ------- | ----------------------------------------------------- | --------------------------------------------------------- |
| **비유**  | **수동 기어 (Manual)**                                    | **자율 주행 (Auto)**                                          |
| **방식**  | **Imperative (명령형)**<br><br>_"A를 하고, B를 해서, C를 만들어라"_ | **Declarative (선언형)**<br><br>_"나는 C가 필요해 (과정은 네가 알아서 해)"_ |
| **데이터** | `Tuple`, `Object`, `Row` (직접 정의)                      | `Table`, `Column`, `Row` (스키마 존재)                         |
| **최적화** | 개발자가 직접 튜닝해야 함                                        | **Query Optimizer**가 알아서 최적화함                             |
| **주사용** | 복잡한 이벤트 제어, 상태 관리                                     | **데이터 분석, ETL, 집계, 리포팅**                                  |

>왜 Table API를 쓰나요?
>**코드량이 1/10로 줍니다.** (윈도우 집계 로직 등이 SQL 한 줄로 끝남)
>**알아서 빨라집니다.** (Optimizer가 비효율적인 연산을 자동으로 고쳐줌)
>**배치와 스트리밍 코드가 똑같습니다.** (코드 수정 없이 모드만 바꾸면 됨)


## 언제 뭘 쓸까?

|**구분**|**Table API & SQL (고수준)**|**DataStream API (저수준)**|
|---|---|---|
|**주 사용처**|**ETL, 대시보드, 통계, 리포팅**|**이상 탐지, 알림 시스템, 실시간 제어**|
|**난이도**|쉬움 (SQL만 알면 됨)|어려움 (Java/Python 객체, State 이해 필요)|
|**제어력**|낮음 (Flink가 알아서 최적화)|**매우 높음 (모든 걸 내가 제어)**|
|**대표 예시**|"지난 1시간 동안의 카테고리별 매출 합계"|"로그인 후 1분 내에 3번 실패 시 계정 잠금"|


---
## 핵심 아키텍처 (Architecture)

Flink의 API는 계층 구조로 되어 있습니다. 
Table API는 DataStream API 위에 얹혀진 **껍데기(Abstraction)** 입니다.

-  **SQL / Table API:** 우리가 작성하는 코드. (가장 쉬움)
- **Planner & Optimizer:** 우리가 쓴 SQL을 가장 효율적인 그래프로 변환.
- **DataStream API:** 변환된 그래프가 실제로 돌아가는 엔진.
- **Runtime:** 물리적인 실행 (Task Manager).

---
## 필수 클래스 3대장 (The Big Three) 

Table API를 시작하려면 딱 3개의 클래스만 알면 됩니다. (코드의 시작점)

### ① `EnvironmentSettings` (설정)

- 이 작업이 **배치(Batch)** 인지 **스트리밍(Streaming)** 인지 결정

```python
from pyflink.table import EnvironmentSettings

# 스트리밍 모드 (Kafka 연결 등) 
settings = EnvironmentSettings.in_streaming_mode()

# 배치 모드 (파일 처리 등)
settings = EnvironmentSettings.in_batch_mode()
```

### ② `TableEnvironment` (통합 관리자)

- `StreamExecutionEnvironment`의 업그레이드 버전입니다.
- 테이블을 만들고(`create`), SQL을 실행하고(`execute_sql`), 설정을 관리합니다.

```python
from pyflink.table import TableEnvironment

t_env = TableEnvironment.create(settings)
```

### ③ `Table` (데이터 그 자체)

- Spark의 **DataFrame**이나 Pandas의 **DataFrame**과 똑같습니다.
- 데이터를 담고 있는 객체이며, `.select()`, `.filter()` 같은 함수를 쓸 수 있습니다.

```python
# SQL 쿼리 결과를 Table 객체로 받음
my_table = t_env.sql_query("SELECT * FROM source_table")

# Table API 함수 사용
result_table = my_table.filter(col("price") > 1000)
```

---
## 코드 비교: "얼마나 쉬워질까?"

**목표:** "사용자별(`user`)로 주문 금액(`amount`) 합계 구하기"

**DataStream API (Hard Mode)**

```python
# 1. Map으로 (user, amount) 추출
# 2. KeyBy로 user별 그룹핑
# 3. Reduce로 amount 더하기
ds.map(lambda x: (x.user, x.amount)) \
  .key_by(lambda x: x[0]) \
  .reduce(lambda a, b: (a[0], a[1] + b[1]))
# (타입 에러 처리하느라 머리 아픔)
```

**Table API (Easy Mode)**

```python
# 그냥 생각하는 그대로 적으면 됨
table.group_by(col("user")) \
     .select(col("user"), col("amount").sum())
```


----
## 배치와 스트리밍의 통합 (Unified API)

이게 Flink Table API의 가장 강력한 무기입니다. 
작성한 코드는 건드리지 않고, `EnvironmentSettings`만 바꾸면 동작이 바뀝니다.

- **Streaming Mode:** 데이터가 들어올 때마다 실시간으로 결과 갱신 (Kafka)
- **Batch Mode:** 저장된 데이터를 한 번에 읽어서 결과 출력 (File, DB)
---
## Flink Table API 파이프라인 실행 순서

코드를 짤 때는 항상 아래 **5단계 순서**를 따릅니다. 순서가 바뀌면 에러납니다.

```scss
① 환경 설정 → ② Source 정의 → ③ 쿼리 작성 → ④ Sink 정의 → ⑤ 실행
```

### ① 환경 설정

```python
from pyflink.table import EnvironmentSettings, TableEnvironment

settings = EnvironmentSettings.in_streaming_mode()
t_env = TableEnvironment.create(settings)
```

### ② Source 테이블 정의 (입력 — 어디서 읽을 건데?)

**실제로 데이터를 읽는 게 아닙니다.** "나중에 여기서 읽을 거야"라는 **설계도 등록**입니다.

```python
t_env.execute_sql("""
    CREATE TABLE source_table (
        order_id     STRING,
        category     STRING,
        total_amount DOUBLE,
        ts           TIMESTAMP(3),
        WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
    ) WITH (
        'connector'                    = 'kafka',
        'topic'                        = 'orders',
        'properties.bootstrap.servers' = 'kafka:9092',
        'properties.group.id'          = 'flink-pipeline-dev',
        'scan.startup.mode'            = 'latest-offset',
        'format'                       = 'json'
    )
""")
```

> WITH 절 옵션 및 CREATE TABLE 상세 설명 → **[[PyFlink_SQL_Integration]]** 페이지 참고

### ③ 쿼리 작성 (Logic — 어떻게 가공할 건데?)

아직 실행이 아닙니다. 쿼리를 **문자열로 들고만 있는 단계**입니다.

```python
agg_query = """
    SELECT 
        window_start,
        window_end,
        category,
        SUM(total_amount) AS total_sales,
        COUNT(order_id)   AS total_orders
    FROM TABLE(
        TUMBLE(TABLE source_table, DESCRIPTOR(ts), INTERVAL '1' MINUTE)
    )
    GROUP BY window_start, window_end, category
"""
```

>윈도우 집계 문법 상세 설명 → **[[PyFlink_SQL_Windows]]** 페이지 참고

### ④ Sink 테이블 정의 (출력 — 어디에 쓸 건데?)

```python
t_env.execute_sql("""
    CREATE TABLE print_sink (
        window_start TIMESTAMP(3),
        window_end   TIMESTAMP(3),
        category     STRING,
        total_sales  DOUBLE,
        total_orders BIGINT
    ) WITH (
        'connector' = 'print'
    )
""")
```

|상황|connector|
|---|---|
|개발/디버깅 중|`'print'`|
|Kafka로 결과 전달|`'kafka'`|
|파일로 저장|`'filesystem'`|

> Sink정의시 주의점 확인하려면  → **[[PyFlink_SQL_Windows#Sink 테이블 선언 시 주의사항]]** 참조 

###  ⑤ 실행 (Execute — 이제 진짜 돌려!)

이 줄이 실행되는 순간 비로소 데이터가 흐르기 시작합니다.

```python
t_env.execute_sql(f"""
    INSERT INTO print_sink
    {agg_query}
""")
```


> **⚠️ `INSERT INTO` 전까지는 아무것도 실행되지 않습니다.**
> ①②③④는 전부 "설계도 그리기"입니다. 
> `INSERT INTO`를 만나는 순간 Flink가 설계도를 보고 Job을 조립해서 실행합니다.


### 전체 흐름 한눈에 보기


```
① t_env 생성
        │
        ▼
② CREATE TABLE source_table (Kafka)    ← 설계도 등록
        │
        ▼
③ agg_query = "SELECT ... TUMBLE ..."  ← 로직 정의 (실행 X)
        │
        ▼
④ CREATE TABLE print_sink              ← 설계도 등록
        │
        ▼
⑤ INSERT INTO print_sink {agg_query}  ← 여기서 진짜 실행!
        │
        ▼
Kafka → Flink → 콘솔 출력
```
