---
aliases:
  - 스파크설치
  - Docker_Spark
  - PySpark설치
  - 로컬환경구축
tags:
  - Spark
  - Docker
related:
  - "[[Spark_Architecture]]"
  - "[[Spark_Concept_Evolution]]"
  - "[[Python_Virtual_Env]]"
  - "[[Dockerfile_Basics]]"
  - "[[Spark_Troubleshooting_FileNotFound]]"
---
## 개념 한 줄 요약

**"내 컴퓨터 안에 가상의 '데이터 센터'를 짓는 과정. (반장 1명, 일꾼 1명, 작업실 1개)"**

* **Master:** 클러스터 관리자 (반장).
* **Worker:** 실제 연산 수행 (일꾼).
* **Jupyter:** 코드를 작성하고 명령을 내리는 곳 (Driver/작업실).

---
## 2. 폴더 구조 (Directory Structure) 

실습을 위해 폴더를 다음과 같이 구성합니다.

```text
apache-spark/             # (루트 폴더)
├── docker-compose.yml    # 설정 파일
├── Dockerfile            # 이미지 설계도
├── requirements.txt      # 파이썬 라이브러리 목록
├── notebooks/            # [Driver 전용] 여기에 .ipynb 파일 저장
└── data/                 # [공유 폴더] 분석할 CSV, Parquet 파일 저장
```

---
## 설정 파일 (Code Core Points)

① `docker-compose.yml`

**핵심:** `spark-worker`는 `notebooks` 폴더가 필요 없습니다. 
대신 `data` 폴더는 반드시 공유해야 합니다!

```YAML
version: '3.8'

services:
  # 1. Master Node (Cluster Manager)
  spark-master:
    build: .
    container_name: spark-master
    command: >
      /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    ports:
      - "9090:8080" # 웹 UI (8080은 충돌이 잦아서 9090 추천)
      - "7077:7077" # 내부 통신용
    networks:
      - spark-net

  # 2. Worker Node (Executor)
  spark-worker:
    build: .
    container_name: spark-worker
    command: >
      /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    depends_on:
      - spark-master
    volumes:
      - ./data:/workspace/data  # [중요] Worker도 데이터를 읽어야 하므로 필수!
    networks:
      - spark-net

  # 3. Jupyter Lab (Driver & Client)
  jupyter:
    build: .
    container_name: spark-jupyter
    command: >
      jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root --NotebookApp.token=''
    volumes:
      - ./notebooks:/workspace # 내 코드 저장소
      - ./data:/workspace/data  # 데이터 저장소
    ports:
      - "8888:8888" # Jupyter 접속
      - "4040:4040" # Spark Job 모니터링 UI
    environment:
      - SPARK_MASTER=spark://spark-master:7077
    depends_on:
      - spark-master
    networks:
      - spark-net

networks:
  spark-net:
    driver: bridgen
```

### ② `Dockerfile`

```Dockerfile
FROM apache/spark:3.5.1

USER root

# 필수 패키지 업데이트 (pip 설치 등)
RUN apt-get update && apt-get install -y python3-pip

# Python 라이브러리 설치
COPY requirements.txt /tmp/requirements.txt
RUN pip3 install --no-cache-dir -r /tmp/requirements.txt \
    && pip3 install jupyterlab

WORKDIR /workspace

# 포트 개방 (Jupyter, SparkUI)
EXPOSE 8888 4040
```

### ③ `requirements.txt`

```text
pandas
numpy
pyarrow
pyspark==3.5.1  # 이미지 버전과 맞추는 것이 좋습니다.
```

---
## 실행 및 접속 방법

**1. 실행: 터미널에서 `apache-spark` 폴더로 이동 후:**

```bash
docker-compose up -d --build
```

**2.상태 확인:**

```bash
docker ps
```

**3.접속 주소:**

- **Jupyter Lab:** `http://localhost:8888` (코드 짜는 곳)
- **Spark Master UI:** `http://localhost:9090` (클러스터 상태 확인)
- **Spark Jobs UI:** `http://localhost:4040` (작업 진행 상황 확인 - 코드 실행 중에만 뜸)

---
## 자주 묻는 질문 (FAQ)

### Q. Worker 폴더에 `notebooks` 폴더 안 만들어도 돼요?

- **네, 안 만들어도 됩니다.**
- 노트북 파일(`.ipynb`)은 **Jupyter(Driver)** 만 가지고 있으면 됩니다. 
- Driver가 코드를 해석해서 Worker에게 "계산 명령(Task)"만 네트워크로 보내기 때문입니다.

### Q. 그럼 Worker에는 왜 `data` 폴더를 연결하나요?

- 코드는 Driver가 보내주지만, **데이터 파일(csv 등)은 보내주지 않기 때문**입니다.
- 코드가 `read.csv("/data/a.csv")`라고 되어 있으면, Worker도 자기 컴퓨터(컨테이너)의 `/data/a.csv` 위치에 파일이 있어야 읽을 수 있습니다.




