---
aliases:
  - SparkSubmitOperator
  - Airflow Spark 연결
  - Spark Connection
tags:
  - Airflow
  - Spark
related:
  - "[[MinIO_Integration]]"
  - "[[Apache_Spark_Setup]]"
---
## 개념 한 줄 요약

**SparkSubmitOperator**는 Airflow가 Spark에게 작업을 시키는 리모컨이야.
특히 **TaskFlow API(`@dag`)** 를 사용하면, 파이썬 함수처럼 깔끔하게 DAG를 정의할 수 있어.

---
##  Pre-requisites: Connection 설정 (UI) 

코드를 돌리기 전에, Airflow에게 **"Spark와 MinIO가 어디 있는지"** 알려줘야 해.
Airflow 웹(http://localhost:8080) -> **Admin** -> **Connections** 메뉴로 가서 2개를 만들어줘.

### ① Spark 연결 (`spark_default`)

* **Connection Id:** `spark_default`
* **Connection Type:** `Spark` (안 보이면 아래 **3. Custom Image 빌드** 참고)
* **Host:** `spark://spark` (Docker 서비스명)
* **Port:** `7077`
* *나머지는 빈칸*

### ② MinIO 감지용 연결 (`minio_default`)

`S3KeySensor`가 파일을 감시하려면 AWS 연결 정보가 필요해.
* **Connection Id:** `minio_default`
* **Connection Type:** `Amazon Web Services`
* **AWS Access Key ID:** `ROOTNAME`
* **AWS Secret Access Key:** `CHANGEME123`
* **Extra:** `{"endpoint_url": "http://minio:9000"}`
    * (⚠️ Extra에 이거 안 적으면 진짜 AWS S3를 찾아가서 에러 남!)

---
## Critical Setup: Custom Image 빌드 (필수)

Airflow 공식 이미지는 최소한의 기능만 있다.
**Spark 연결 플러그인** 과 **Java(JDK)** 가 없어서 실행이 불가능하다.
따라서 `Dockerfile`을 만들어 새로 빌드해야 한다.

### Step 1. `requirements.txt` 추가 

```text
apache-airflow-providers-apache-spark 
apache-airflow-providers-amazon
```


### Step 2. `Dockerfile` 작성 (Java 17 설치 포함)

**중요:** Airflow 컨테이너가 Spark Executor와 통신하려면 **반드시 Java가 설치되어 있어야 한다.**

```Dockerfile
# 사용하는 Airflow 버전에 맞춤
FROM apache/airflow:3.1.6

USER root

# 1. OpenJDK 17 설치 (Spark 통신 필수)
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
         openjdk-17-jre-headless \
  && apt-get autoremove -yqq --purge \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# Java 홈 환경변수 설정
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-arm64

USER airflow

# 2. Provider 패키지 설치
COPY requirements.txt /requirements.txt
RUN pip install --no-cache-dir -r /requirements.txt
```

### Step 3. `docker-compose.yaml` 수정

이미지를 다운로드(`image:`) 받는 대신, 내 설정대로 빌드(`build: .`)하도록 변경.

```YAML
x-airflow-common:
  &airflow-common
  # image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.10.4}  <-- 주석 처리
  build: .                                             # <-- 현재 폴더 Dockerfile 사용
```

### Step 4. 적용

```bash
docker-compose down
docker-compose up -d --build
```

---
## Code Template (Standard Pattern)

모든 Spark 작업에 통용되는 **표준 DAG 패턴**이다.

### [원칙] 지휘자(DAG)와 연주자(App) 분리

- **DAG 파일:** Airflow 스케줄링 및 설정 주입 담당. (`dags/my_dag.py`)
- **App 파일:** 실제 Spark 데이터 처리 로직 담당. (`dags/my_app.py`)
    - _DAG 파일 안에 Spark 로직을 섞지 말 것!_

> [[Airflow_Spark_MinIO_Pipeline_Build]] 참고 

---
## Troubleshooting 

### Q. `Task exited with return code 1`

- **원인:** 대부분 **파일 경로** 문제.
- **해결:** `application` 경로(`/opt/airflow/...`)가 실제 컨테이너 내부에 존재하는지 확인.
    

### Q. `Connection Refused` / 무한 대기

- **원인:** Airflow(Driver)와 Spark(Executor)가 서로 통신을 못 함.
- **해결:** `spark.driver.host`를 `host.docker.internal`로 설정했는지, `spark.driver.port`를 고정했는지 확인.

### Q. `ClassNotFoundException: org.apache.hadoop.fs.s3a.S3AFileSystem`

- **원인:** Airflow 쪽에 S3 통신용 JAR 파일이 없음.
- **해결:** `SparkSubmitOperator`의 `packages` 파라미터에 `hadoop-aws`와 `aws-java-sdk-bundle`을 반드시 명시해야 함.

----
## 내가만난 Troubleshooting: Connection Type에 'Spark'가 안 떠요! (Custom Image 빌드) 

Airflow를 처음 설치하면 **Connections** 메뉴에 `Spark`나 `Amazon Web Services`가 없을 수 있어. 
기본 이미지는 "깡통" 상태라서 플러그인(Provider)을 따로 설치해서 **"나만의 이미지"** 를 구워야 해.

### Step 1. 쇼핑 리스트 작성 (`requirements.txt`)

`apache-airflow` 폴더 안에 파일을 만들고 필요한 라이브러리를 적어.

```txt
apache-airflow-providers-apache-spark 
apache-airflow-providers-amazon
```

### Step 2. 요리 레시피 작성 (`Dockerfile`)

똑같이 `apache-airflow` 폴더 안에 파일을 만들어. 
(버전은 `docker-compose.yaml`과 맞춰야 함!)

```Dockerfile
# 1. 베이스 이미지 가져오기
# 이 버전은 docker-comose.yaml에서 x-airflow-common: 있던 기존 이미지랑 버전을 맞춰야함!!!!
FROM apache/airflow:3.1.6  

# 2. 파일 복사 & 설치
COPY requirements.txt /requirements.txt
RUN pip install --no-cache-dir -r /requirements.txt
```

### Step 3. 주방 설정 변경 (`docker-compose.yaml`)

이제 "다운로드(image)" 받지 말고 "직접 요리(build)" 하라고 설정을 바꿔야 해.

```YAML
x-airflow-common:
  &airflow-common
  # image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:3.1.6}  <-- 1. 기존 이미지 주석 처리 (#)
  build: .                                             # <-- 2. 현재 폴더(Dockerfile)로 빌드하라고 지시
```

### Step 4. 빌드 및 실행 (Build & Run) 🚀

터미널에서 `--build` 옵션을 줘서 이미지를 새로 만들어야 적용돼.

```bash
docker-compose down
docker-compose up -d --build
```


---
### 💡 Tip: 재시작할 때 `-v` 옵션 주의! (Data Persistence)

이미지를 새로 빌드하거나 설정을 바꾸고 재시작할 때, 습관적으로 `-v`를 붙이지 마!

* **`docker-compose down`**: 컨테이너만 삭제. (DB, Connection 정보, 로그 **유지됨** ✅) -> **평소에 쓰는 것**
* **`docker-compose down -v`**: 볼륨까지 삭제. (DB 초기화, Connection 정보 **삭제됨** ❌) -> **완전 초기화할 때만 사용**

> **상황:** 우리는 방금 UI에서 Spark Connection을 만들었기 때문에, `-v`를 쓰면 설정이 날아가서 또 만들어야 해. 그냥 `down`만 쓰자!
