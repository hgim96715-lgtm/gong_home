---
aliases:
  - Spark S3 연동
  - Spark MinIO 연결
  - hadoop-aws
  - S3AFileSystem
tags:
  - Spark
  - MinIO
related:
  - "[[Apache_Spark_Setup]]"
  - "[[MinIO_Docker_Setup]]"
---
## 개념 한 줄 요약

Spark가 MinIO(S3)와 대화하려면 **"통역사(Library)"** 가 필요해.
기본적으로 Spark는 HDFS(하둡 파일 시스템)만 알아듣기 때문에, **`hadoop-aws`** 라는 라이브러리를 추가하고 설정을 만져줘야 해.

---
## Pre-requisites (네트워크 이슈 해결) 

우리는 지금 **Airflow(MinIO)** 와 **Spark**를 **서로 다른 `docker-compose` 파일**로 띄웠어. 
즉, 서로 **다른 네트워크(남남)** 라서 통신이 안 돼.

가장 쉬운 해결책은 **`host.docker.internal`** 을 쓰는 거야.
* **의미:** "도커 컨테이너 밖의 **내 진짜 컴퓨터(Host)** 를 통해서 옆집(MinIO)을 찾아가라!"
* **MinIO 주소:** `http://minio:9000` (X) 👉 **`http://host.docker.internal:9000`** (O)

---
##  Configuration (설정 파일 수정) 

**Step 1.** `spark/spark_conf/spark-defaults.conf` 파일을 열어.
**Step 2.** 아래 내용을 그대로 복사해서 붙여넣어. (주석 참고!)

- `spark-defaults.conf`는 **"공통 설정을 미리 세팅해두는 곳"**

```properties
# ---------------------------------------------------------
# 1. Event Logging (사용자님이 가져온 로그 설정 - 살려둠!)
# ---------------------------------------------------------
spark.eventLog.enabled           true
spark.eventLog.dir               file:/tmp/spark-events
spark.history.fs.logDirectory    file:/tmp/spark-events

# ---------------------------------------------------------
# 2. MinIO / S3 Connection (이게 없으면 MinIO 연결 불가! 🚨)
# ---------------------------------------------------------
# [중요] Spark가 켜질 때 AWS 연결용 라이브러리를 자동으로 받아옴
spark.jars.packages              org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262

# MinIO 접속 주소 (Docker 간 통신을 위해 host.docker.internal 사용)
spark.hadoop.fs.s3a.endpoint          http://host.docker.internal:9000

# MinIO 계정 정보 (Docker Compose에서 설정한 값과 같아야 함)
spark.hadoop.fs.s3a.access.key        ROOTNAME
spark.hadoop.fs.s3a.secret.key        CHANGEME123

# 기타 필수 설정
spark.hadoop.fs.s3a.path.style.access true
spark.hadoop.fs.s3a.impl              org.apache.hadoop.fs.s3a.S3AFileSystem
spark.hadoop.fs.connection.ssl.enabled false

# (선택) 로컬 실행 시 Ivy 캐시 경로 고정
spark.driver.extraJavaOptions         -Divy.cache.dir=/tmp/ivy2 -Divy.home=/tmp/ivy2
```

---
## Execution & Verification (테스트)

설정을 바꿨으니 **Spark를 재시작**해야 적용돼!

### Step 1. 재시작

```bash
cd spark
docker-compose down
docker-compose up -d
```

### Step 2. 테스트 코드 작성

(`spark/apps/word_count.py`) MinIO에 파일을 썼다가 읽어오는 코드를 작성해 보자.

```python
from pyspark.sql import SparkSession

# 스파크 세션 생성
spark = SparkSession.builder \
    .appName("MinIO Integration Test") \
    .getOrCreate()

# 1. 간단한 데이터 생성
data = [("Samsung", 1000), ("LG", 800), ("SK", 500)]
columns = ["Company", "Stock"]
df = spark.createDataFrame(data, columns)

# 2. MinIO에 쓰기 (Write)
# s3a://{버킷이름}/{경로}
print("Writing data to MinIO...")
df.write \
    .mode("overwrite") \
    .csv("s3a://airflow-bucket/test_data")

# 3. MinIO에서 읽기 (Read)
print("Reading data from MinIO...")
read_df = spark.read.csv("s3a://airflow-bucket/test_data", header=False, inferSchema=True)
read_df.show()

print("Success! 🎉")
spark.stop()
```

### Step 3. Execution (실행 명령어 상세)

단순히 코드만 치는 게 아니라, **"내가 지금 누구에게, 어디로 보내라고 명령하는지"** 정확히 알고 쳐야 해.

#### **1. 컨테이너 이름 확인 (Pre-check)** 

먼저 내 스파크 마스터의 진짜 이름을 알아야 해. (docker-compose가 자동으로 지어준 이름일 수 있어)

```bash
docker ps
# NAMES 컬럼 확인: spark-master 인지 spark-spark-1 인지 체크!
# (아래 명령어는 'spark-spark-1' 기준이야)
```

#### **2. 실행 명령어 (Run)** 

이 명령어는 **"MinIO에 있는 `input.txt`를 읽어서, MinIO의 `output` 폴더에 결과를 저장해라"** 라는 뜻이야.

```bash
docker exec -it spark-spark-1 spark-submit \
  --master spark://spark:7077 \
  /opt/bitnami/spark/work/word_count.py \
  s3a://airflow-minio/input.txt \
  s3a://airflow-minio/output
```

#### 명령어 해부 (Detailed Analysis)

- `docker exec -it spark-spark-1`: 
	- **누구에게?** `spark-spark-1` 컨테이너(컴퓨터) 안에 들어가서 명령을 내려라.

- `spark-submit`:
	- **무엇을?** 스파크 작업을 제출(Submit)하겠다.

- `/opt/bitnami/spark/work/word_count.py`:
	- **어떤 코드로?** 컨테이너 내부의 이 경로에 있는 파이썬 파일을 실행해라. (우리가 로컬에서 마운트 해둔 파일)

- `s3a://airflow-minio/input.txt`:
	- **입력 데이터:** (중요 ) `s3a://`가 붙었으니 **MinIO(원격 창고)** 에서 가져와라.

- `s3a://airflow-minio/output`:
	- **출력 데이터:** (중요 ⭐️) `s3a://`가 붙었으니 **MinIO(원격 창고)** 에 저장해라.
	- _만약 `s3a://`를 빼고 `/opt/...`라고 적으면 MinIO가 아니라 컨테이너 내부 서랍장에 저장되니 주의!_

### Step 4. Verification (결과 확인)

터미널에서 에러 없이 끝났다면, 이제 웹 화면에서 **"작업 성공 여부"** 와 **"결과물"** 을 직접 확인해 보자.

#### **① Spark Master UI (작업 성공 여부)**

- **주소:** [http://localhost:8081](https://www.google.com/search?q=http://localhost:8081) (⚠️ 8080 아님!)
- **확인 방법:**
    1. 화면 중간의 **Completed Drivers** (또는 Running Drivers) 표를 봐.
    2. **App Name**이 `WordCount`이고, **State**가 **`FINISHED`** 로 되어 있으면 연산 성공! 
    3. _(만약 `ERROR`나 `FAILED`라면 클릭해서 `stderr` 로그를 확인해야 해)_

#### **② MinIO UI (결과물 저장 확인)**

- **주소:** [http://localhost:9001](https://www.google.com/search?q=http://localhost:9001)
- **확인 방법:**
    1. **Object Browser** 메뉴 클릭.
    2. **`airflow-minio`** (버킷) 클릭.
    3. **`output`** 폴더가 새로 생겼는지 확인.
    4. 폴더 안에 **`_SUCCESS`** 파일(성공 깃발)과 **`part-00000...csv`** 파일(실제 데이터)이 있다면 완벽하게 저장된 거야! 

----
## Best Practice: Config 관리의 정석 (Hard Coding vs Config File) 💎

인터넷에서 남의 코드를 보면 `spark = SparkSession.builder.config(...).config(...).getOrCreate()` 처럼 코드가 엄청 긴 경우가 있어.
그건 **"환경 설정 파일(`spark-defaults.conf`)을 미리 세팅하지 않았기 때문"** 이야.

### ① 하드 코딩 (Hard Coding) - ❌ Bad Practice

코드 안에 설정값(ID, PW, 주소)을 직접 때려 박는 방식.
* **단점 1:** 코드가 지저분해져서 비즈니스 로직(데이터 처리)이 안 보임.
* **단점 2:** MinIO 비밀번호가 바뀌면 모든 파이썬 파일을 다 열어서 고쳐야 함. (유지보수 지옥)
* **단점 3:** 깃허브(GitHub)에 올릴 때 비밀번호가 전 세계에 노출됨. 😱

```python
# 남들의 코드 (설정이 덕지덕지)
spark = SparkSession.builder \
    .config("fs.s3a.access.key", "ROOTNAME") \
    .config("fs.s3a.secret.key", "CHANGEME123") \
    .config("fs.s3a.endpoint", "http://minio:9000") \
    .getOrCreate()
```

### ② 설정 파일 분리 (External Configuration) - ✅ Good Practice

우리는 `spark-defaults.conf`라는 **"전역 설정 파일"** 에 공통 설정을 미리 다 넣어뒀어.
- **장점:** 파이썬 코드에는 **"스파크 줘!"** 한 마디만 하면 됨. 알아서 설정 파일을 읽어옴.
- **결과:** 코드가 훨씬 **심플하고 우아(Elegant)** 해짐.

```python
# 우리의 코드 (깔끔 그 자체 ✨)
spark = SparkSession.builder.appName("MyJob").getOrCreate()
```

> **결론:** `spark-defaults.conf`는 **"공통 설정을 미리 세팅해두는 곳"** 이야.
>  덕분에 매번 `.config()`를 주렁주렁 달지 않아도 돼!

---
## Critical Reminder: 실행 전 필수 체크 (The "Two-Terminal" Rule) ⚠️

우리는 지금 **"본부(Airflow+MinIO)"**와 **"작업장(Spark)"**을 분리해서 운영하고 있어.
그래서 작업을 할 때는 반드시 **두 개의 터미널**에서 각각 도커를 켜줘야 해!

### ❌ 흔한 실수 (One Side Only)

* **Spark만 켜고 코드를 돌리면?** * 👉 `Connection refused` 에러 발생! (저장할 창고인 MinIO가 꺼져 있으니까)
* **Airflow(MinIO)만 켜고 기다리면?** * 👉 아무 일도 안 일어남 (일할 사람인 Spark가 출근을 안 했으니까)

### ✅ 올바른 실행 순서 (Both Sides Up)

**Terminal 1 (본부 켜기):**

```bash
cd ~/gong_study_de/apache-airflow
docker-compose up -d  # MinIO (창고) Open 🏠
```

**Terminal 2 (작업장 켜기):**

```bash
cd ~/gong_study_de/apache-airflow/spark
docker-compose up -d  # Spark (요리사) 출근 ⚡️
```

>**한 줄 요약:** 
>식당 문(MinIO)도 열고, 요리사(Spark)도 출근시켜야 요리가 나옵니다! 
>둘 다 켜져 있는지 `docker ps`로 항상 확인하세요.


