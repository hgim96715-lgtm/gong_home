---
aliases:
  - guide
  - Cassandra
tags:
  - Spark
  - Cassandra
related:
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Spark_Streaming_Cassandra_Setup]]"
  - "[[Spark_Streaming_Code_Analysis]]"
---
## 1단계: 카산드라 데이터 준비 (Data Preparation)

먼저 정적 데이터(Static Data)인 유저 정보를 카산드라에 만들어둡니다.

### ① 카산드라 접속 (cqlsh)

터미널을 열고 아래 명령어를 입력해 카산드라 컨테이너에 접속합니다.

```bash
# 방법 1: 바로 접속하기 (추천)
docker exec -it [CASSANDRA_CONTAINER_NAME] cqlsh -u [ID] -p [PASSWORD]
# (기본값 예시: docker exec -it cassandra cqlsh -u cassandra -p cassandra)


# 방법 2: 컨테이너 들어간 후 접속하기
# docker exec -it cassandra bash
# cqlsh -u cassandra -p cassandra
```

- **성공 확인:** 터미널 프롬프트가 `cqlsh>` 로 바뀌면 접속 성공입니다!

### ② 키스페이스(DB) 및 테이블 생성

`cqlsh>` 프롬프트에 아래 내용을 복사해서 붙여넣으세요. (세미콜론 `;` 필수!)

```sql
-- 1. Keyspace(DB) 생성
CREATE KEYSPACE IF NOT EXISTS [KEYSPACE_NAME]
WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};

-- 2. 생성한 DB로 이동 (필수!)
USE [KEYSPACE_NAME];
```

- `{sql}CREATE KEYSPACE test_db` 
	- **의미:** "test_db라는 이름의 공간(Keyspace)을 만들어라."
	- **설명:** MySQL이나 Oracle에서의 **`CREATE DATABASE`** 와 같습니다. 테이블들을 담는 가장 큰 폴더를 만드는 겁니다.
- `IF NOT EXISTS`
	- **의미:** "만약 없을 때만 만들어라."
- `{sql}WITH replication = { ... }`
	- 카산드라는 데이터가 날아가는 것을 방지하기 위해 여러 서버에 데이터를 **복제(Replication)** 해서 저장하는 기능이 핵심입니다.그 설정을 여기서 합니다
	- `{sql}'class': 'SimpleStrategy'` 
		- **의미:** "복잡한 네트워크 설정 없이, 그냥 단순하게 복사해."
		- **언제 쓰나요?** 개발용, 테스트용, 또는 데이터센터가 1개일 때 씁니다. (지금처럼 Docker 하나 띄워놓고 할 때는 무조건 이거 씁니다.)
		- 실제 운영 환경에서는 `NetworkTopologyStrategy`라는 걸 써서 "서울 리전에 2개, 부산 리전에 1개 저장해" 같이 복잡하게 설정합니다.
	- `{sql}replication_factor': 1`
		- **의미:** "복사본(Replica)을 **1개**만 유지해."
		- Docker 컨테이너를 **딱 1개**(`cassandra`)만 띄웠죠? 서버가 한 대뿐이라 복사본을 2개, 3개 만들고 싶어도 둘 곳이 없습니다. 그래서 `1`로 설정하는 겁니다.
		- 실제 회사에서는 서버가 여러 대라서 보통 `3`으로 설정합니다. 서버 하나가 죽어도 데이터가 살아있도록요.

### ③ 테이블 생성 및 더미 데이터 입력

스파크에서 조인할 테이블을 만듭니다. **컬럼 이름과 타입은 스파크 코드와 일치해야 합니다.**
조인 테스트를 위해 미리 유저 정보를 넣어둡니다.

```sql
-- 3. 테이블 생성
CREATE TABLE IF NOT EXISTS [TABLE_NAME] (
    [COLUMN_1] [TYPE] PRIMARY KEY, 
    [COLUMN_2] [TYPE],
    [COLUMN_3] [TYPE]
);

-- 4. 테스트용 데이터 입력 (Optional)
INSERT INTO [TABLE_NAME] ([COL1], [COL2], [COL3]) VALUES ('[VAL1]', '[VAL2]', '[VAL3]');

-- 확인
SELECT * FROM [TABLE_NAME];
```

**예시**

```sql
-- 3. 테이블 생성
CREATE TABLE IF NOT EXISTS users (
    login_id text PRIMARY KEY, 
    user_name text, 
    last_login timestamp
);

INSERT INTO users (login_id, user_name, last_login) VALUES ('100', 'Kim', '2023-01-01 00:00:00');
INSERT INTO users (login_id, user_name, last_login) VALUES ('101', 'Lee', '2023-01-01 00:00:00');
INSERT INTO users (login_id, user_name, last_login) VALUES ('102', 'Park', '2023-01-01 00:00:00');

-- 잘 들어갔나 확인
SELECT * FROM users;
```

---
## 2단계: 스파크 코드 실행 (Spark Submit) 

이제 준비된 파이썬 코드(`kafka_static_join.py`)를 실행합니다. 
카산드라 커넥터와 카프카 패키지를 함께 로딩해야 합니다.

### 실행 명령어 (터미널)

스파크 컨테이너 내부(`/opt/spark`)에서 실행한다고 가정합니다.

```bash
/opt/spark/bin/spark-submit \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:[SPARK_VERSION],com.datastax.spark:spark-cassandra-connector_2.12:[CONNECTOR_VERSION] \
  --conf spark.cassandra.connection.host=[CASSANDRA_CONTAINER_NAME] \
  [YOUR_PYTHON_FILE].py
```

**예시**

```bash
/opt/spark/bin/spark-submit \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1,com.datastax.spark:spark-cassandra-connector_2.12:3.5.0 \
  --conf spark.cassandra.connection.host=cassandra \
  kafka_static_join.py
```

- **실행 후:** 에러 없이 `Batch: 0` 같은 로그가 뜨면서 멈춰있으면(대기 중이면) 성공입니다!

---
## 3단계: 테스트 및 결과 확인 (Test) 

스파크가 켜진 상태에서 **새 터미널**을 열고 데이터를 보내봅니다.

###  ① Kafka에 데이터 보내기 (Producer)

**주의:** 파이썬 코드의 `.option("subscribe", "[TOPIC_NAME]")`과 동일한 토픽이어야 합니다.

```bash
# 카프카 프로듀서 실행
# Docker 안의 Kafka 컨테이너에 명령을 보냅니다.
/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka:9092 --topic [TOPIC_NAME]
# 예시 /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka:9092 --topic casan

# 메시지 입력 (JSON) -> 입력 후 엔터!
> {"[TIME_COL]":"2026-02-04 19:30:00", "[ID_COL]": "[ID_VALUE]"}
# 예시> {"create_date":"2026-02-04 19:30:00","login_id": "100"}
```

### ② 스파크 화면 확인 (Console)

스파크가 실행 중인 터미널에 아래와 같이 결과가 뜨는지 확인합니다.

```text
+--------+---------+-------------------+
|login_id|user_name|         last_login|
+--------+---------+-------------------+
|     100|      Kim|2026-02-04 19:30:00|
+--------+---------+-------------------+
```


### ③ 카산드라 데이터 확인 (DB Check)

다시 `cqlsh`로 가서 데이터가 진짜로 바뀌었는지 봅니다.

```sql
SELECT * FROM [TABLE_NAME] WHERE [PK_COL] = '[ID_VALUE]';
--예시SELECT * FROM users WHERE login_id = '100';
```

- **성공 기준:** `last_login` 시간이 방금 보낸 시간(`19:30:00`)으로 바뀌어 있으면 완벽 성공!

---
## Cassandra 길라잡이 (Cheatsheet)

"지금 내가 어디 있는지, 테이블은 뭐가 있는지" 궁금할 때 쓰는 명령어 모음입니다.

### 내가 만든 DB(Keyspace)들이 뭐 뭐 있지?

```sql
DESCRIBE KEYSPACES;
```

- 결과에 만든 (예)`test_db`가 보여야 합니다.

### 특정 DB로 들어가고 싶어 (이동)

```sql
USE [KEYSPACE_NAME];
```

- 프롬프트가 `cqlsh:[KEYSPACE_NAME]>`로 바뀌어야 제대로 들어간 겁니다.
- **[매우 중요] 반드시 `USE` 명령어를 먼저 실행해야 합니다!**
	- 이걸 안 하고 `SELECT`를 하면 **"테이블이 없다(Table not found)"** 거나 **"키스페이스가 지정되지 않았다(No keyspace specified)"** 는 에러가 뜹니다.
	- 접속할 때마다 습관적으로 **`USE [KEYSPACE_NAME];`** 를 가장 먼저 쳐주세요!

### 이 DB 안에 테이블이 뭐 뭐 있지? (테이블 목록)

SQL의 `SHOW TABLES`와 다릅니다!

```sql
DESCRIBE TABLES;
```

### 테이블 구조(컬럼)를 보고 싶어

```sql
DESCRIBE TABLE [TABLE_NAME];
```

- `login_id`, `user_name` 같은 만든 컬럼 정보가 쫘르륵 나옵니다.

### 화면이 너무 지저분해! (청소)

```bash
cls  # 또는 clear
```



