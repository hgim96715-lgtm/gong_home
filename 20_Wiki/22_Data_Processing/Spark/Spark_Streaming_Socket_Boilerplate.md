---
aliases:
  - Streaming Template
  - Socket Boilerplate
  - 스트리밍 템플릿
tags:
  - Spark
  - Streaming
related:
  - "[[Spark_Streaming_Intro]]"
---
##  개요 ️

이 코드는 **Netcat(소켓)** 으로 들어오는 데이터를 실시간으로 받아서 처리하는 가장 기본적인 구조입니다.

---
## 실행 준비 (Netcat 서버 설치 및 실행)

스파크를 돌리기 전에, 먼저 데이터를 보내줄 **"가짜 서버"** 를 켜야 합니다.

- **Docker 환경**을 사용 중이라면 `netcat`이 없을 수 있습니다.
- 먼저 터미널(Jupyter Terminal 등)을 열고 **설치부터 실행까지** 아래 순서대로 진행하세요.

### ① Netcat 설치 (최초 1회)

Docker 컨테이너 안에는 보통 `nc` 명령어가 없으므로 설치해야 합니다.

```bash
# 1. 패키지 목록 업데이트 
apt update

# 2. netcat 설치 
apt-get install netcat
```

### ② 가짜 서버 실행 (매번 실행)




```bash
# 터미널에서 포트 9999번 열기
nc -lk 9999
```

#### 참고: `nc` (Netcat) 명령어가 뭔가요?

- **정의:** 리눅스/맥에 기본으로 깔려 있는 **"네트워크의 맥가이버 칼"** 입니다.
- **용도:** 스파크 공부할 때 거창한 Kafka를 설치하는 대신, **"내 터미널을 채팅 서버처럼"** 만들어서 테스트할 때 씁니다.
- 옵션 해부:
	- **`-l` (Listen):** "누가 접속할 때까지 귀 기울이고 있어라" (서버 모드).
	- **`-k` (Keep-alive):** "한 번 연결이 끊겨도 죽지 말고 계속 살아 있어라" (필수).
	- **`9999`:** 사용할 문(Port) 번호.


---
## 만능 템플릿 코드 (Python)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *

# ==========================================
# 1. 설정 (Configuration)
# ==========================================
APP_NAME = "UniversalStreamingApp"
HOST = "localhost"
PORT = 9999

# 출력 모드 선택 (중요!)
# - "append": 새로운 데이터가 들어올 때마다 그 데이터만 출력 (로그 수집, 필터링)
# - "complete": 전체 집계 결과를 매번 갱신해서 출력 (Count, GroupBy)
# - "update": 바뀐 데이터만 출력
OUTPUT_MODE = "append" 

# ==========================================
# 2. 세션 생성 & 입력 (Input Source)
# ==========================================
spark = SparkSession.builder.appName(APP_NAME).getOrCreate()

# 소켓에서 데이터 읽기 (무한한 DataFrame 생성)
# 기본 컬럼명은 'value' (String) 하나입니다.
raw_df = spark.readStream \
    .format("socket") \
    .option("host", HOST) \
    .option("port", PORT) \
    .load()

# ==========================================
# 3. 비즈니스 로직 (Transformation) - [TODO: 여기를 수정하세요!]
# ==========================================

# [시나리오 1] 들어온 그대로 보여주기 (Raw)
processed_df = raw_df

# [시나리오 2] JSON 데이터 파싱하기 (실무 패턴)
# 예: 터미널에 {"name": "gong", "age": 25} 입력 시
# schema = StructType().add("name", StringType()).add("age", IntegerType())
# processed_df = raw_df.select(from_json(col("value"), schema).alias("data")).select("data.*")

# [시나리오 3] 특정 단어가 포함된 로그만 필터링
# processed_df = raw_df.filter(col("value").contains("ERROR"))

# [시나리오 4] 단어 개수 세기 (집계) -> OUTPUT_MODE를 "complete"로 바꿔야 함!
# processed_df = raw_df.groupBy("value").count()

# ==========================================
# 4. 출력 (Output Sink)
# ==========================================
print(f"🚀 Streaming Job '{APP_NAME}' is running on {HOST}:{PORT}...")

query = processed_df.writeStream \
    .outputMode(OUTPUT_MODE) \
    .format("console") \
    .option("truncate", "false") \
    .start()

query.awaitTermination()
```

---
## 코드 상세 해설

### ① `spark.readStream` (수도꼭지 연결하기)

```python
raw_df = spark.readStream \
    .format("socket") \
    .option("host", HOST) \
    .option("port", PORT) \
    .load()
```

- **`readStream`**: 스파크에게 "이건 끝난 파일이 아니라, **계속 들어오는 데이터**야!"라고 알려줍니다. 
	(배치 처리의 `read`와 다름)
- **`format("socket")`**: 데이터가 어디서 오나요? -> "네트워크 소켓(TCP)에서 옵니다." (Netcat과 연결)
- **`load()`**: 실제 연결을 준비합니다. 이때 만들어진 `raw_df`는 데이터가 **무한히 추가되는 테이블**처럼 동작합니다.

### ② `writeStream ... start()` (수도꼭지 틀기)

```python
query = processed_df.writeStream \
    .outputMode("append") \
    .format("console") \
    .start()
```

- **`writeStream`**: 가공된 데이터를 어디론가 내보낼 준비를 합니다.
- **`outputMode(...)`**: 데이터를 어떻게 보여줄지 결정합니다. **(아주 중요!)**
	- `append`: **새로운 데이터**만 보여줍니다. (단순 로그 수집)
	- `complete`: **전체 결과**를 다시 계산해서 보여줍니다. (`count` 같은 집계할 때 필수)
	- `update`: **바뀐 내용**만 보여줍니다.
- **`format("console")`**: 결과를 화면(콘솔)에 출력합니다. (나중엔 `kafka`, `parquet` 등으로 바뀝니다)
- **`start()`**: **"자, 이제 진짜 실행해!"** (Action). 이 명령어가 실행되어야 비로소 데이터가 흐르기 시작합니다.

### ③ `query.awaitTermination()` (계속 지켜보기)

```python
query.awaitTermination()
```

- 이게 왜 필요한가요?
	- 스트리밍 작업은 `start()`를 하면 **백그라운드(별도 스레드)** 에서 돌아갑니다.
	- 만약 이 줄이 없다면? 파이썬 스크립트는 "어? 할 일 다 했네?" 하고 **바로 종료(Exit)** 되어 버립니다. 스파크도 같이 꺼지겠죠.
- **역할:** "사용자가 강제로 끄기 전까지는, 메인 프로그램이 죽지 말고 **계속 기다려라(Block)**" 라는 뜻입니다. 즉, 서버를 계속 켜두는 역할을 합니다.

---
## 응용 가이드 (Scenario) 

- **로그만 보고 싶을 때:** `OUTPUT_MODE = "append"` (기본값)
- **개수를 세고 싶을 때 (Word Count):**
    1. 로직을 `processed_df = raw_df.groupBy("value").count()` 로 변경.
    2. 반드시 `OUTPUT_MODE = "complete"` 로 변경. (안 그러면 에러 남!)
