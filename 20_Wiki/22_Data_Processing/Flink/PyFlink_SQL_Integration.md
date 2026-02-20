---
aliases:
  - WITH
  - execute_sql
  - CREATE
  - SQL
  - PyFlink
tags:
  - PyFlink
related:
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_Table_API_Intro]]"
  - "[[PyFlink_Table_Expressions]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[PyFlink_SQL_Windows]]"
  - "[[Python_Database_Connect]]"
---
## 한줄 요약

**"Flink와 바깥 세상을 연결하는 다리."**
(표준 SQL 문법(`CREATE TABLE`)에 `WITH` 절을 추가하여 Kafka, DB, File 등 외부 시스템과 연결하는 방법)


---
## `execute_sql()`: Flink에게 SQL 던지기

TableEnvironment의 가장 기본이 되는 함수입니다.
DDL(테이블 생성), DML(데이터 조회/입력), DQL(쿼리) 모두 이 함수 하나로 처리합니다.

```python
# 1. 테이블 생성 (DDL)
t_env.execute_sql("CREATE TABLE ...")

# 2. 데이터 조회 및 출력 (DQL)
t_env.execute_sql("SELECT * FROM ...").print()

# 3. 데이터 입력 (DML) -> 실제 Job 제출
t_env.execute_sql("INSERT INTO sink_table SELECT ...")
```

>**Tumbling Window 집계 쿼리**
>Window 집계 쿼리 작성법(TUMBLE, HOP, SESSION 문법, `window_start`/`window_end` 자동 생성, `DESCRIPTOR` 사용법 등)은 **[[PyFlink_SQL_Windows]]** 페이지를 참고하세요.

---
## `CREATE TABLE`의 해부학

Flink SQL의 테이블 정의는 크게 **두 부분**으로 나뉩니다.

```sql
CREATE TABLE my_table (
    -- [Part 1] 스키마 정의 (데이터의 생김새)
    user_id STRING,
    price INT,
    ts TIMESTAMP(3)
) WITH (
    -- [Part 2] 커넥터 설정 (연결 정보 & 옵션) 
    'connector' = 'kafka',
    'topic' = 'logs',
    ...
)
```

|Part|역할|한 줄 설명|
|---|---|---|
|**Part 1** (Schema)|논리적 구조|컬럼 이름과 타입 — 데이터가 어떻게 생겼는가|
|**Part 2** (Options)|물리적 연결|커넥터 설정 — 데이터가 어디 있고 어떻게 읽는가|

### 컬럼은 전부 다 적어야 할까? ❌ NO

Python 딕셔너리에 컬럼이 50개 있어도, **Flink SQL에서 쓸 것만 골라 적으면 됩니다.**
Flink는 정의되지 않은 컬럼은 조용히 무시하고, 적어둔 컬럼만 파싱합니다.

#### 단, 필수로 적어야하는 컬럼

**`GROUP BY`에 쓰는 컬럼**

```sql
GROUP BY user_id -- 반드시 스키마에 선언
```

**`SUM()`, `COUNT()`, `AVG()` 등 집계 대상 컬럼**

```sql
SUM(price)   -- price 선언 필수
COUNT(order_id)  -- order_id 선언 필수
```

**윈도우 집계에 쓰는 시간 컬럼 + Watermark**

```sql
ts TIMESTAMP(3),
WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
-- TUMBLE, HOP 등 시간 윈도우를 쓰려면 시간 컬럼은 필수
```

**`WHERE`나 `JOIN` 조건에 쓰는 컬럼**

```sql
WHERE status = 'SUCCESS'  -- status 선언 필수
```

### **⚠️ 이름(Spelling) 완벽히 일치시키기**

>Flink는 Python/Kafka에서 넘어온 데이터의 **필드 이름을 그대로 매핑**합니다.
>오타 하나가 `null` 지옥을 만듭니다.

```sql
-- ❌ 잘못된 예시 — 값이 전부 null로 들어옴
ts          TIMESTAMP(3),   -- 이름 다름
totalPrice  INT             -- 카멜케이스 vs 스네이크케이스 불일치
 
-- ✅ 올바른 예시
event_timestamp TIMESTAMP(3),
total_price     INT
```

> **특히 시간 컬럼 주의!** `timestamp`, `event_time`, `created_at` 등 프로젝트마다 이름이 다르고, 오타 내도 에러 없이 그냥 `null`로 통과되어 나중에 집계가 전부 깨집니다.

### **🔴 예약어 충돌 — 백틱으로 감싸기**

>SQL에는 **예약어(Reserved Keyword)** 가 있습니다. 
>컬럼 이름이 예약어와 겹치면 Flink가 데이터가 아닌 **SQL 문법으로 오해**해서 파싱 에러가 납니다.

```sql
-- ❌ 위험 — timestamp는 SQL 예약어
timestamp TIMESTAMP(3)

-- ✅ 안전 — 백틱으로 감싸면 "이건 컬럼 이름이야"라고 명시
`timestamp` TIMESTAMP(3)
```

자주 충돌하는 대표적인 예약어들

| 충돌 위험 컬럼명   | 이유                     |
| ----------- | ---------------------- |
| `timestamp` | SQL 타입 키워드             |
| `time`      | SQL 타입 키워드             |
| `date`      | SQL 타입 키워드             |
| `value`     | SQL 함수 키워드             |
| `order`     | SQL 절 키워드 (`ORDER BY`) |

>컬럼 이름을 바꿀 수 없는 상황이라면 **항상 백틱을 습관화**하는 게 가장 안전합니다.

### 타입 매칭 — Python ↔ Flink 타입 맞추기

Python에서 보낸 타입과 Flink 선언 타입이 맞지 않으면 **파싱 실패 또는 null**이 됩니다.

|Python 타입|Flink 타입|비고|
|---|---|---|
|`int` (작은 수)|`INT`|약 21억 이하|
|`int` (큰 수)|`BIGINT`|21억 초과 시 반드시 `BIGINT`|
|`float`|`DOUBLE`|`FLOAT`보다 `DOUBLE` 권장 (정밀도)|
|`str`|`STRING`||
|`bool`|`BOOLEAN`||
|`str` (날짜/시간 형식)|`TIMESTAMP(3)`|포맷이 ISO 8601이어야 자동 파싱됨|
|`dict` / `list`|`STRING` (JSON 문자열로 처리)|Flink에 직접 dict 타입 없음|
|`str(uuid.uuid4())`|`STRING`|UUID는 결국 문자열 — `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"` 형태로 전달됨|
|`random.choice(문자열 리스트)`|`STRING`|리스트 요소가 문자열이면 STRING, 숫자면 INT — **리스트 안 값의 타입**을 보고 결정|

>**`INT` vs `BIGINT` 실수 주의!**
> 매출 합산, 누적 클릭 수처럼 숫자가 커질 수 있는 컬럼은 처음부터 `BIGINT`로 선언하세요. 
> `INT` 범위를 초과하면 **오버플로우로 음수가 튀어나오는** 황당한 버그가 생깁니다.

### ⚠️ 시간 컬럼 JSON 파싱 옵션

`str` → `TIMESTAMP(3)` 매핑 시, Python이 보내는 시간 문자열 형식에 따라 WITH 절 옵션을 추가해야 합니다.

|Python 코드|결과 형식|Flink 파싱 여부|
|---|---|---|
|`datetime.now().strftime('%Y-%m-%d %H:%M:%S')`|`2024-01-01 12:00:00`|✅ 기본값으로 파싱됨|
|`datetime.now().isoformat()`|`2024-01-01T12:00:00.123456`|❌ 파싱 실패 → null|

`T`가 들어가는 ISO-8601 형식이라면 WITH 절에 아래 옵션을 추가해야 합니다.

```sql
WITH (
    'connector'                      = 'kafka',
    'format'                         = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'  -- ← 이거 추가!
)
```

>**안 붙이면 에러도 안 납니다.** 조용히 `null`로 들어와서 윈도우 집계가 전부 `0`으로 나오는 황당한 상황이 생깁니다.


### 한눈에 정리


```
Kafka 필드 100개
        │
        ▼
   내가 쓸 것만 선언  ◀── GROUP BY / SUM / WHERE / JOIN 기준으로 추려내기
        │
        ▼
   이름 철자 Python과 100% 일치 확인
        │
        ▼
   CREATE TABLE 완성
```


---
## `WITH` 절 완전 정복 (Connector Options)

`WITH` 절에는 커넥터 종류에 따라 수십 가지 옵션이 들어갈 수 있습니다. 가장 중요한 3가지 카테고리로 정리해 드립니다.

### ① 연결 대상 (`connector`, `topic`, `server`)

**"누구랑, 어디서, 어떻게 연결할 건데?"** 를 지정하는 옵션들입니다.

|옵션|예시 값|한 줄 설명|
|---|---|---|
|`connector`|`'kafka'`|카프카와 연결하겠다|
|`topic`|`'user_logs'`|카프카의 `user_logs` 토픽을 읽겠다|
|`properties.bootstrap.servers`|`'kafka:9092'`|카프카 서버의 IP 주소 + 포트 번호|
|`properties.group.id`|`'my-group'`|카프카에 제출하는 **소속 팀 신분증**|

#### `group.id` 상세 설명

`group.id`는 카프카에게 **"나는 어느 팀에서 온 누구야"** 라고 밝히는 신분증입니다. 
카프카는 이 ID를 기준으로 **"이 팀은 마지막으로 100번째 데이터까지 읽었지"** 를 기억하고 이어서 데이터를 전달합니다.

>**⚠️ 치명적 실수 주의**
>실시간 대시보드(Streamlit)와 집계 파이프라인(Flink)의 `group.id`를 **똑같이 지으면 안 됩니다.**
>두 애플리케이션이 데이터를 서로 뺏어가면서, 대시보드는 뚝뚝 끊기고 Flink 집계는 반토막이 납니다.

**권장 네이밍 규칙** `[프로젝트명]-[애플리케이션명]-[환경]`

```TOML
# Flink용
project-flink-aggregator-dev

# Streamlit용
project-streamlit-dashboard-dev
```


### ② 데이터 형식 (`format`)

"데이터가 어떤 모양으로 포장되어 있어?"를 지정합니다.

- `'format' = 'json'`: 데이터가 JSON 문자열이야. (`{"id": "a", "price": 10}`)
- `'format' = 'csv'`: 콤마로 구분된 텍스트야. (`a,10`)
- `'format' = 'avro'`: Avro 바이너리 포맷이야. (별도 라이브러리 필요)

### ③ 시작 위치 (`scan.startup.mode`)

"과거 데이터부터 읽을까, 지금부터 읽을까?" (Kafka 전용)

| **옵션 값**               | **의미**          | **비유**                                |
| ---------------------- | --------------- | ------------------------------------- |
| **`earliest-offset`**  | **가장 처음부터**     | "책 처음부터 정주행해." (모든 과거 데이터 처리)         |
| **`latest-offset`**    | **지금 들어오는 것부터** | "지금부터 방송되는 것만 봐." (실시간 데이터만 처리)       |
| **`group-offsets`**    | **마지막 읽은 곳부터**  | "읽던 페이지(책갈피)부터 이어서 봐." (중단 후 재개 시 사용) |
| **`specific-offsets`** | **특정 위치부터**     | "50페이지부터 봐."                          |

---
## 자주 쓰는 Connector

`'connector' = '...'` 부분에 들어갈 수 있는 대표적

### Kafka (`'connector' = 'kafka'`)

가장 많이 씁니다. 실시간 데이터 파이프라인의 표준입니다.

```sql
WITH (
    'connector' = 'kafka',
    'topic' = 'clicks',
    'properties.bootstrap.servers' = 'kafka:9092',
    'format' = 'json'
)
```

### print (`'connector' = 'print'`)

**디버깅용**으로 최고입니다. 결과를 Flink 로그(Console)에 찍어줍니다.

```sql
WITH (
    'connector' = 'print'
)
```

###  Datagen (`'connector' = 'datagen'`)

**테스트 데이터 생성용**입니다. Kafka 없이 로직을 테스트하고 싶을 때 가짜 데이터를 마구 만들어줍니다.

```sql
WITH (
    'connector' = 'datagen',
    'rows-per-second' = '10' -- 초당 10개 생성
)
```

### FileSystem (`'connector' = 'filesystem'`)

파일(CSV, Parquet)을 읽거나 쓸 때 사용합니다.

```sql
WITH (
    'connector' = 'filesystem',
    'path' = 'file:///tmp/output',
    'format' = 'csv'
)
```

###  jdbc (`connector='jdbc'`)

관계형 데이터베이스(PostgreSQL, MySQL, Oracle 등)의 데이터를 읽거나, 분석된 결과를 DB에 밀어 넣을(Write) 때 사용합니다. 
실시간 대시보드와 연결할 때 가장 많이 쓰이는 커넥터입니다.

```sql
WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:postgresql://postgres:5432/my_database',  -- 1. DB 주소
    'table-name' = 'sales_aggregation',                     -- 2. 대상 테이블명
    'username' = 'my_user',                                 -- 3. DB 접속 계정
    'password' = 'my_password',                             -- 4. DB 비밀번호
    'driver' = 'org.postgresql.Driver'                      -- 5. DB 통역사(드라이버) 지정
)
```

- **`'url'`**: 연결할 DB의 주소입니다. 사용하는 DB 종류에 따라 형식이 다릅니다.
	- PostgreSQL 예시: `jdbc:postgresql://호스트:포트/데이터베이스명` > 데이터베이스명은 내가 원하는 걸로 정의 
	- MySQL 예시: `jdbc:mysql://호스트:포트/데이터베이스명`

- **`'table-name'`**: 데이터를 집어넣거나 읽어올 실제 DB 속 **테이블 이름**입니다. 
	- (주의: Flink가 테이블을 대신 만들어주지 않으므로, DB에 미리 테이블을 생성해 두어야 합니다.)

- **`'username'` / `'password'`**: 데이터베이스에 접근하기 위한 아이디와 비밀번호입니다.
- **`'driver'`**: (선택이지만 권장) Flink가 해당 DB와 통신할 때 사용할 자바 클래스 이름입니다. 명시해 주는 것이 안전합니다.
	- PostgreSQL: `org.postgresql.Driver`
	- MySQL: `com.mysql.cj.jdbc.Driver`


**필수 준비물 (매우 중요!)** JDBC 커넥터를 사용하려면 반드시 **2개의 JAR 파일**을 Flink의 `lib` 폴더에 넣거나 파이썬 코드에서 로드해야 합니다.

1. **Flink JDBC 커넥터 JAR** (예: `flink-connector-jdbc-*.jar`)
2. **해당 DB의 JDBC 드라이버 JAR** (예: `postgresql-*.jar` 또는 `mysql-connector-java-*.jar`)

```bash
# JobManager에 다운로드
docker exec -it -u root project-jobmanager-1 curl -fL -o /opt/flink/lib/flink-connector-jdbc-3.1.2-1.18.jar https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar
docker exec -it -u root project-jobmanager-1 curl -fL -o /opt/flink/lib/postgresql-42.6.0.jar https://repo1.maven.org/maven2/org/postgresql/postgresql/42.6.0/postgresql-42.6.0.jar

# TaskManager(실제 일꾼)에 다운로드
docker exec -it -u root project-taskmanager-1 curl -fL -o /opt/flink/lib/flink-connector-jdbc-3.1.2-1.18.jar https://repo1.maven.org/maven2/org/apache/flink/flink-connector-jdbc/3.1.2-1.18/flink-connector-jdbc-3.1.2-1.18.jar
docker exec -it -u root project-taskmanager-1 curl -fL -o /opt/flink/lib/postgresql-42.6.0.jar https://repo1.maven.org/maven2/org/postgresql/postgresql/42.6.0/postgresql-42.6.0.jar

# 설치된 JAR 파일을 인식하도록 Flink 컨테이너 재시작
docker restart project-jobmanager-1 project-taskmanager-1
```

_(주의: `docker-compose down`을 하면 파일이 날아가므로 반드시 `restart`만 해야 합니다.)_