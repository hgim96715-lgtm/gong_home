---
aliases:
  - SparkSubmitOperator 설정
  - Airflow Spark MinIO 연동
tags:
  - Airflow
  - Spark
  - MinIO
  - Troubleshooting
related:
  - "[[Spark_MinIO_Integration]]"
  - "[[Airflow_Spark_Integration]]"
---
# Airflow + Spark + MinIO 로컬 파이프라인 구축 성공기

## 개념 한 줄 요약

**"지휘자(Airflow)가 일꾼(Spark)에게 창고(MinIO)의 데이터를 처리하라고 명령하는 구조."**
로컬 Docker 환경에서 `SparkSubmitOperator`를 사용하여 S3 호환 스토리지인 MinIO의 데이터를 읽고 쓰는 완전한 ETL 파이프라인.

---
## 왜 필요한가 (Why)

* **문제:** Spark를 단독으로 쓰면 스케줄링이 어렵고, 로컬에서 Docker 컨테이너끼리 통신할 때 네트워크(IP/Port) 문제와 라이브러리(JAR) 부재로 연결이 자주 끊김.
* **해결:**
    1.  **Airflow**로 작업 의존성과 스케줄 관리.
    2.  **`host.docker.internal`** 과 **고정 포트(20000)** 를 사용하여 컨테이너 간 통신 벽을 뚫음.
    3.  **`hadoop-aws` JAR 패키지**를 강제 주입하여 Spark가 S3(MinIO) 프로토콜을 이해하게 만듦.

---
## Infrastructure Setup (Docker Compose 수정)

코드에서 포트를 설정하기 전에, **물리적인 도커 컨테이너의 문(Port)** 을 먼저 열어줘야 한다. 
이걸 안 하면 Spark가 작업을 마치고 Airflow에게 보고하러 올 때 문이 잠겨 있어 **"All masters are unresponsive"** 에러가 발생한다.

```yaml
services: airflow-worker: 
# ... (기존 설정) ... 
ports: - "8793:8793" 
# Airflow 기본 포트 
# 👇 [필수 추가] Spark Driver 통신용 포트 개방 
- "20000:20000" # Spark Driver 
- "20001:20001" # Block Manager Port
```

**Tip:** 설정 변경 후에는 반드시 `docker-compose down && docker-compose up -d`로 재시작해야 적용된다.


---
##  Environment Setup (환경 설정 필수!)

코드를 돌리기 전에 **Airflow Worker 컨테이너**가 Spark를 실행할 수 있는 상태여야 한다.
(이게 안 되어 있으면 `Java not found` 또는 `InvalidClassException` 발생)

### 터미널 명령어 (일회성 설정). > 만약 설치가 안되어있다면 

```bash
# 1. Airflow Worker에 Java 17 설치 (Spark 통신용)
docker exec -u 0 -it apache-airflow-airflow-worker-1 bash -c "apt-get update && apt-get install -y openjdk-17-jre-headless"

# 2. PySpark 버전 일치시키기 (Spark 컨테이너와 동일한 3.5.3 버전)
# (권한 문제 발생 시 /home/airflow 소유권 변경 선행 필요)
docker exec -it apache-airflow-airflow-worker-1 python -m pip install pyspark==3.5.3
```

---
## Code Core Points (성공 코드)

### ① 지휘자: Airflow DAG (`spark_word_count.py`)

Spark에게 **"누가, 어디서, 어떻게"** 일해야 하는지 모든 설정을 주입하는 것이 핵심.

```python
from airflow.decorators import dag
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
import pendulum

default_args = {
    "owner": "airflow",
    'start_date': pendulum.datetime(2026, 1, 20, tz="Asia/Seoul"),
    "retries": 1,
}

@dag(
    default_args=default_args,
    schedule=None, 
    catchup=False,
    tags=['spark', 'minio'],
)
def spark_word_count_taskflow():

    # [Point 1] MinIO에 파일이 들어왔는지 감시
    check_minio_file = S3KeySensor(
        task_id='check_minio_file',
        bucket_name='airflow-minio',
        bucket_key='input.txt',
        aws_conn_id='minio_default',
        poke_interval=10,
        timeout=60 * 5
    )

    # [Point 2] Spark 작업 제출 (여기가 핵심!)
    spark_task = SparkSubmitOperator(
        task_id="spark_submit_task",
        conn_id="spark_default",
        application='/opt/airflow/dags/word_count_app.py', # 로직 파일 분리 필수
        
        # [핵심 1] S3(MinIO) 통신을 위한 필수 라이브러리 다운로드
        packages="org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262",
        
        # [핵심 2] 네트워크 및 인증 설정 강제 주입
        conf={
            # Driver(Airflow)가 Executor(Spark)와 통신하기 위한 길 뚫기
            "spark.driver.bindAddress": "0.0.0.0",
            "spark.driver.host": "host.docker.internal", # Mac Docker 통신 마법의 단어
            "spark.driver.port": "20000",                # 고정 포트 사용
            "spark.blockManager.port": "20001",
            
            # MinIO 연결 정보
            "spark.hadoop.fs.s3a.endpoint": "http://minio:9000",
            "spark.hadoop.fs.s3a.access.key": "ROOTNAME",
            "spark.hadoop.fs.s3a.secret.key": "CHANGEME123",
            "spark.hadoop.fs.s3a.path.style.access": "true",
            "spark.hadoop.fs.s3a.impl": "org.apache.hadoop.fs.s3a.S3AFileSystem",
            "spark.hadoop.fs.connection.ssl.enabled": "false",
        },
        
        # [핵심 3] Airflow 컨테이너 내의 Java 위치 지정 (M1/M2 Mac 기준 arm64)
        env_vars={"JAVA_HOME": "/usr/lib/jvm/java-17-openjdk-arm64"},
        
        application_args=[
            "s3a://airflow-minio/input.txt", 
            "s3a://airflow-minio/output_airflow"
        ]
    )

    check_minio_file >> spark_task

spark_dag = spark_word_count_taskflow()
```

### ② 일꾼: Spark App (`word_count_app.py`)

여기에는 **Airflow 관련 코드가 없어야 함**. 순수하게 데이터 처리 로직만 존재.

```python
import sys
from pyspark.sql import SparkSession

if __name__ == "__main__":
    # SparkSession 생성 (설정은 spark-submit 명령어로 이미 다 넘어옴)
    spark = SparkSession.builder.appName("WordCountApp").getOrCreate()

    if len(sys.argv) < 3:
        sys.exit("Usage: word_count_app.py <input> <output>")

    input_path = sys.argv[1]
    output_path = sys.argv[2]

    try:
        # S3a 경로에서 읽기
        df = spark.read.text(input_path)
        
        # Word Count 로직
        counts = df.rdd.flatMap(lambda x: x.value.split(" ")) \
                       .map(lambda x: (x, 1)) \
                       .reduceByKey(lambda a, b: a + b)
                       
        # 결과 저장
        counts.saveAsTextFile(output_path)
        print("DEBUG: Success!")
        
    except Exception as e:
        print(f"DEBUG: Error: {e}")
        raise e
    finally:
        spark.stop()
```

---
## Detailed Analysis (왜 이렇게 설정했는가?)

**`packages="org.apache.hadoop:hadoop-aws..."`**

- Spark는 기본적으로 S3(`s3a://`)를 모른다. 이 옵션을 줘야 실행 시점에 Maven Repository에서 관련 JAR 파일을 받아와서 통역사 역할을 시킨다. 
- 이게 없으면 `ClassNotFoundException: S3AFileSystem` 에러가 난다.

**`park.driver.host": "host.docker.internal"`**

- Airflow(Driver)와 Spark(Worker)가 서로 다른 컨테이너에 있다. 
- Spark가 작업을 끝내고 Airflow에게 보고하러 올 때, `localhost`라고 하면 자기 자신을 찾게 된다. 호스트(Mac)를 경유해서 찾아오라고 명시하는 설정이다.

**`env_vars={"JAVA_HOME": ...}`**

- `SparkSubmitOperator`는 내부적으로 자바 명령어를 쓴다. 
- Airflow 이미지에는 자바가 기본 설치되어 있지 않으므로, 우리가 수동 설치한 경로를 알려줘야 한다.

---
## Common Beginner Misconceptions (주의사항)

- **❌ DAG 파일 안에 Spark 로직을 다 넣으면 된다?**
    - 절대 안 됨. 
    - DAG 파일은 Airflow가 읽는 것이고, 실제 작업은 Spark 클러스터로 전송되어야 한다. 
    - 그래서 `word_count_app.py`를 따로 분리해야 한다.

- **❌ Airflow 컨테이너 설정만 잘하면 된다?**
    - 아님. Airflow 컨테이너 내부의 **Java 설치 여부**와 **PySpark 라이브러리 버전(Cluster와 일치)** 이 가장 중요한 숨겨진 조건이다.
        
- **❌ `s3://` 쓰면 되는 거 아냐?**
    - Hadoop 생태계에서는 S3를 쓸 때 `s3a://` 스킴을 사용해야 속도와 호환성이 보장된다.

