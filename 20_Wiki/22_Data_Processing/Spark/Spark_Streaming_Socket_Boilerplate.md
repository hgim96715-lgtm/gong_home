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
  - "[[Linux_File_Types]]"
  - "[[00_Apache_Spark_HomePage]]"
  - "[[Streaming_Source_Comparison]]"
linked:
  - file:///Users/gong/gong_study_de/apache-spark/notebooks/step22.ipynb
---
##  개요

**"복사해서 로직만 바꾸면 끝나는 만능 스트리밍 템플릿."**

이 코드는 **Structured Streaming(DataFrame)** 방식을 사용하여, 소켓(Socket)으로 들어오는 실시간 데이터를 처리하는 가장 표준적인 구조입니다.

---
## 실행 준비 (Docker 사용자 전용 가이드) 

Docker를 사용 중이라면 **새로운 터미널(Host)** 을 열고 아래 순서대로 접속해서 서버를 켜야 합니다.

### ① 실행 중인 컨테이너 찾기 & 접속

먼저 Jupyter나 Spark가 돌고 있는 컨테이너의 ID를 찾아야 합니다.

```bash
# 1. 실행 중인 컨테이너 목록 확인
docker ps

# 2. 목록에서 'jupyter' 또는 'spark'가 포함된 컨테이너의 [CONTAINER ID] 복사
# 예: a1b2c3d4e5f6

# 3. 해당 컨테이너 내부로 접속 (bash 실행)
docker exec -it <CONTAINER_ID> bash
# 예: docker exec -it a1b2c3d4e5f6 bash
```

### ② Netcat 설치 및 실행 (컨테이너 내부)

접속에 성공했다면(프롬프트가 바뀜), 이제 `nc`를 설치하고 실행합니다.

```bash
# 4. 패키지 목록 업데이트 (최초 1회)
apt-get update

# 5. netcat 설치 (최초 1회)
apt-get install -y netcat

# 6. 포트 9999번 열기 (스파크가 접속할 때까지 대기)
nc -lk 9999
```

(커서가 깜빡거리면 성공입니다! 이 창을 끄지 말고 두세요.)

---
## 만능 템플릿 코드 (Python)

```python
from pyspark.sql import SparkSession
import pyspark.sql.functions as F
import pyspark.sql.types as t

# ==========================================
# 1. 설정 (Configuration)
# ==========================================
APP_NAME = "UniversalStreamingApp"
HOST = "localhost"
PORT = 9999

# 출력 모드 선택 (중요!)
# - "append": 새로운 데이터만 출력 (로그 수집, 단순 필터링)
# - "complete": 전체 집계 결과를 매번 갱신 (Count, GroupBy 필수)
# - "update": 바뀐 데이터만 출력
OUTPUT_MODE = "append" 

# ==========================================
# 2. 세션 생성 & 입력 (Input Source)
# ==========================================
spark = SparkSession.builder.appName(APP_NAME).getOrCreate()

# [핵심 1] 읽기 시작 (readStream)
raw_df = spark.readStream \
    .format("socket") \
    .option("host", HOST) \
    .option("port", PORT) \
    .load()

# ==========================================
# 3. 비즈니스 로직 (Transformation) - [TODO: 여기를 수정하세요!]
# ==========================================

# [기본] 들어온 그대로 보여주기
processed_df = raw_df

# [응용] 예: 특정 단어가 포함된 로그만 필터링
# processed_df = raw_df.filter(col("value").contains("ERROR"))

# [집계] 예: 단어 개수 세기 (OUTPUT_MODE="complete"로 변경 필수!)
# processed_df = raw_df.groupBy("value").count()

# ==========================================
# 4. 출력 (Output Sink)
# ==========================================
print(f"🚀 Streaming Job '{APP_NAME}' is running on {HOST}:{PORT}...")

# [핵심 2] 쓰기 시작 (writeStream)
query = processed_df.writeStream \
    .outputMode(OUTPUT_MODE) \
    .format("console") \
    .option("truncate", "false") \
    .start()

# [핵심 3] 대기 (Await)
query.awaitTermination()
```

---
## 코드 상세 해설 (초보자용 가이드) 

위 코드에서 가장 중요한 3가지 부분의 의미를 하나씩 뜯어봅시다.

### ① spark.readStream (수도꼭지 연결하기)

```python
raw_df = spark.readStream \
    .format("socket") \
    .option("host", HOST) \
    .option("port", PORT) \
    .load()
```

- **`readStream`**: 스파크에게 "이건 끝난 파일이 아니라, **계속 들어오는 데이터**야!"라고 알려줍니다.
- **`format("socket")`**: 데이터 소스를 지정합니다. (여기선 방금 켠 Netcat 서버)
- **`load()`**: 연결을 준비하고 **무한한 테이블(DataFrame)** 을 만듭니다.

>[[Streaming_Source_Comparison]] 참고 

### ② `writeStream ... start()` (수도꼭지 틀기)

```python
query = processed_df.writeStream \
    .outputMode("append") \
    .format("console") \
    .option("truncate", "false") \
    .start()
```

- **`writeStream`**: 가공된 데이터를 내보낼 준비를 합니다.
- **`outputMode(...)`**: 데이터를 어떻게 보여줄지 결정합니다.
    
    - `append`: 새로운 줄만 출력 (로그).
    - `complete`: 전체 결과를 갱신해서 출력 (집계).
        
- **`format("console")`**: 결과를 화면(콘솔)에 찍습니다.
- **`option("truncate", "false")`**: **"말 줄임표(...) 금지!"** 
    - 긴 로그가 들어왔을 때 뒷부분이 잘리지 않고 **전체 내용을 다 보여줍니다.** 디버깅 필수 옵션입니다.
- **`start()`**: **"실행!"** (Action). 이 명령어가 있어야 데이터가 흐르기 시작합니다.

### ③ `query.awaitTermination()` (계속 지켜보기)

```python
query.awaitTermination()
```

- **역할:** "사용자가 강제로 끄기(`Stop`) 전까지는 프로그램이 죽지 말고 **계속 기다려라(Block)**" 라는 뜻입니다.
- 이게 없으면 스크립트가 실행되자마자 바로 종료되어 버립니다.
---
## Output Mode 3대장 완벽 정리 

`writeStream.outputMode(...)`에 들어갈 옵션은 데이터의 성격에 따라 결정됩니다.

### ① Append Mode (기본값) - "로그 쌓기"

* **동작:** 마지막 배치 이후에 **새로 추가된 데이터만** 출력합니다
* **특징:**
    * 과거 데이터는 건드리지 않고, 오직 **증분(Incremental)** 데이터만 관심 있을 때 씁니다
    * 데이터가 계속 늘어나는(Ever-growing) 로그성 데이터에 적합합니다
* **주의:** `GroupBy` 같은 집계(Aggregation)를 할 때는 기본적으로 사용할 수 없습니다. (과거 데이터를 모르니까 합계를 낼 수 없음).

### ② Update Mode - "변경된 것만 알림"
* **동작:** 새로운 데이터 + **값이 바뀐(Updated) 데이터만** 출력합니다
* **특징:**
    * 배치 간에 **상태(State)** 를 유지해야 할 때 유용합니다
    * 예: "실시간 누적 합계(Running Total)"나 "활성 사용자 목록"처럼 시간이 지나면서 값이 변하는 경우.
* **차이점:** 값이 안 바뀐 데이터는 출력하지 않으므로 Complete 모드보다 가볍습니다.

### ③ Complete Mode - "전체 덮어쓰기"

* **동작:** 매 배치마다 **전체 결과(All records)** 를 싹 다 다시 출력합니다
* **특징:**
    * 새로운 데이터뿐만 아니라 업데이트된 데이터까지 포함한 **전체 집합(Entire Set)** 을 보고 싶을 때 씁니다
    * 데이터 양이 많아지면 **리소스(Resource)를 많이 잡아먹습니다** 매번 전체를 다시 계산해서 보여줘야 하니까요.
* **필수 상황:** `GroupBy().count()` 같은 **단순 집계**를 할 때는 무조건 이 모드를 써야 합니다.

---

### ⚡️ 요약표 (언제 뭘 쓸까?)

| 모드 | 상황 | 비유 |
| :--- | :--- | :--- |
| **Append** | 단순 필터링, 로그 수집 | **"새로 온 편지만 우체통에 넣어줘."** |
| **Update** | 상태 변화 감지 | **"점수가 바뀐 학생 성적표만 다시 가져와."** |
| **Complete** | 전체 통계/집계 (Count, Sum) | **"전교생 성적표를 처음부터 끝까지 다시 출력해와."** |