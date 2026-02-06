---
aliases:
  - Flink Docker 설치
  - Flink 로컬 환경 구축
  - PyFlink Docker Setup
tags:
  - Flink
  - Docker
related:
  - "[[Flink_Execution_Models]]"
  - "[[00_Apache Flink_HomePage]]"
linked:
  - https://nightlies.apache.org/flink/flink-docs-stable/docs/deployment/resource-providers/standalone/docker/
---
## 개요 (Overview)

Flink는 기본적으로 Java/Scala 프로그램입니다. 
파이썬(PyFlink)을 돌리려면 **"기본 Flink 이미지에 Python을 설치한 커스텀 이미지"** 가 필요합니다.
우리는 개발용(Session)과 배포용(Application) 두 가지 환경을 모두 준비합니다.

---
##  폴더 구조 (Directory Structure)

 아래 3개 파일을 만드세요.

```text
        ├── Dockerfile                     # (1) 이미지 빌드 설계도
        ├── docker-compose.session.yml     # (2) 개발용 (우리가 쓸 거)
        └── docker-compose.application.yml # (3) 배포용 (참고용)
        └── playground/ # (3) 파이썬 코드 저장소 (새로 생성!)
		        └── src/ 
			        └── 파일명.py
```

---
## 파일 작성 (Configuration)

### ① Dockerfile

> **목표:** "Flink 2.2.0 베이스에 Python 3.10과 PyFlink를 설치해줘."

```Dockerfile
# [1] 베이스 이미지 선택
# 이미 Flink 엔진(Java/Scala)이 설치된 공식 이미지를 가져옵니다.
FROM flink:2.2.0-scala_2.12

# [2] 관리자 권한 획득
# 패키지 설치(apt-get)를 위해 잠시 root 유저로 전환합니다.
USER root

# [3] Python 3 런타임 설치 (핵심!)
# - python3: 파이썬 실행기
# - python3-pip: 파이썬 패키지 관리자
# - ca-certificates, curl: 보안 인증서 및 다운로드 도구
# - ln -s: 'python'이라고만 쳐도 'python3'가 실행되게 연결
# - rm -rf: 설치 후 찌꺼기 파일 삭제 (이미지 용량 다이어트)
RUN apt-get update && \
    apt-get install -y \
        python3 \
        python3-pip \
        ca-certificates \
        curl && \
    ln -s /usr/bin/python3 /usr/bin/python && \
    rm -rf /var/lib/apt/lists/*

# [4] 최소 의존성 설치
# - apache-flink 라이브러리를 통째로 깔지 않고, 실행에 꼭 필요한 보조 라이브러리만 설치합니다.
# - typing_extensions: 파이썬 버전 간 타입 호환성을 맞춰주는 필수 라이브러리
# - --break-system-packages: 최신 리눅스의 설치 차단 기능을 우회하는 옵션
RUN pip3 install --no-cache-dir --break-system-packages \
    typing_extensions

# [5] 환경 변수 설정 (명찰 달기)
# Flink 시스템에게 "파이썬 실행 파일은 여기에 있어!"라고 위치를 알려줍니다.
ENV PYTHON_EXECUTABLE=/usr/bin/python3
ENV PYFLINK_CLIENT_EXECUTABLE=/usr/bin/python3

# [6] 권한 복귀 및 작업 폴더 설정
# 보안을 위해 다시 일반 유저(flink)로 돌아오고, 작업 시작 위치를 잡습니다.
USER flink
WORKDIR /opt/flink

```

- 도커 이미지(`flink:2.2.0`) 안에는 이미 Flink의 핵심 엔진(Java/Scala)이 들어있다. 

### ② docker-compose.session.yml (개발용)

> **목표:** "반장(JobManager)과 일꾼(TaskManager)을 띄워놓고 대기해. 내가 필요할 때마다 일감 던질게."

- **핵심:** `playground` 폴더를 연결해서, 내 컴퓨터에서 코드를 고치면 컨테이너에도 바로 반영되게 함.

```yaml
services:
  # 👑 JobManager (상시 대기)
  jobmanager:
    build: .
    image: pyflink:1.18.1
    platform: linux/amd64 # 👈 [추가] "나 인텔 컴퓨터야!"라고 속임
    volumes:
	  - ./playground:/opt/flink/playground # 코드 공유 폴더 연결
    ports:
      - "8081:8081" # 호스트:8081 -> 컨테이너:8081 연결 (대시보드용)
    command: jobmanager
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2

  # 👷 TaskManager (일꾼)
  taskmanager:
	image: pyflink:1.18.1
	platform: linux/amd64 # 👈 [추가] 여기도 똑같이!
    volumes:
		- ./playground:/opt/flink/playground # 일꾼도 코드를 볼 수 있어야 함
    depends_on:
      - jobmanager
    command: taskmanager
    scale: 1
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2
```

|Mac|기본 아키텍처|
|---|---|
|M1 / M2 / M3|`linux/arm64`|
|Intel Mac|`linux/amd64`|
인데, `PyFlink`에서 linux/arm64 이걸로 하면 에러,오류가 너무나서 `linux/amd64` 이걸로 속임 

### ③ docker-compose.application.yml (배포용)

> **목표:** "이 코드(my_job.py)만을 위해 클러스터를 띄우고, **일 끝나면 알아서 해산해.**"

```yaml
version: "2.2"
services:
  # =========================================
  # 👑 JobManager (Application Mode)
  # =========================================
  jobmanager:
    # [1] 이미지: 아까 Dockerfile로 만든 '파이썬이 설치된 이미지'를 써야 합니다.
    image: pyflink:2.2.0
    
    # [2] 포트: 내 컴퓨터(localhost) 8081로 접속하면 컨테이너 8081(대시보드)로 연결
    ports:
      - "8081:8081"
      
    # [3] 명령어 (핵심!): 
    # "standalone-job"은 Flink 클러스터를 띄우면서 동시에 잡을 실행합니다.
    # Java일 땐 --job-classname을 쓰지만, Python은 --python 옵션으로 파일을 지정합니다.
    # (주의: /opt/flink/usrlib/my_job.py 파일이 실제로 있어야 실행됩니다)
    command: standalone-job --python /opt/flink/usrlib/my_job.py
    
    # [4] 볼륨 (코드 연결):
    # 내 컴퓨터의 './src' 폴더를 컨테이너의 '/opt/flink/usrlib'로 연결합니다.
    # 즉, src 폴더에 'my_job.py'를 넣어두면 Flink가 그걸 읽어서 실행합니다.
    volumes:
      - ./src:/opt/flink/usrlib
      
    # [5] 환경 변수:
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager  # 반장 이름 (서비스명)
        parallelism.default: 2              # 기본 병렬성 (슬롯 2개 다 써라)

  # =========================================
  # 👷 TaskManager
  # =========================================
  taskmanager:
    image: pyflink:2.2.0
    depends_on:
      - jobmanager
    command: taskmanager
    scale: 1
    
    # [중요!] Application Mode에서는 일꾼(TM)도 코드를 가지고 있어야 합니다.
    # 반장이 "이 코드 실행해!"라고 던져주는데, 일꾼한테 그 파일이 없으면 에러 납니다.
    volumes:
      - ./src:/opt/flink/usrlib
      
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        taskmanager.numberOfTaskSlots: 2   # 일꾼 한 명이 슬롯 2개 가짐
        parallelism.default: 2
```

- `application.yml`을 지금 당장 실행(`up`)하면 **에러**가 나면서 바로 꺼질 겁니다.
- **이유:** `command`에 적힌 **`my_job.py`** 파일이 아직 없기 때문입니다.
- **해결:** 나중에 우리가 파이썬 코드를 다 짜고 나서, 그걸 배포 테스트할 때 이 파일을 사용하게 될 겁니다. 지금은 **"아하, 배포할 땐 이렇게 설정하는구나"** 하고 저장만 해두시면 됩니다!

---
## 실행 방법 (How to Run)

### 1. Session Mode 실행 (개발 시)

```bash
# 개발할 땐 이거 쓰세요!
docker compose -f compose.session.yml up -d --build
```

### 2. Application Mode 실행 (배포 테스트 시)

```bash
# 나중에 배포 테스트할 때만 쓰세요 (지금은 src 폴더가 없어서 에러 날 수 있음)
docker compose -f compose.aplication.yml up -d --build
```

- `-f`는 **"File (파일)"** 의 약자
- 원래 `docker-compose` 명령어는 아무 말 안 하면 **"기본 이름"** 인 `docker-compose.yml`이라는 파일만 찾습니다.
- **기본 파일 말고 내가 지정하는 이 파일(`-f`)을 써!"** 라고 콕 집어 알려주는 옵션

### 확인 

http://localhost:8081/#/overview 접속 후 Task Slots가 2개인지 확인.

---
## 실행 및 테스트 

**playground/src/...py 파일을 하나 만든 후**

### 컨테이너 내부 접속(로그인)

```bash
docker compose -f compose.session.yml exec jobmanager bash
```

- (프롬프트가 `flink@...:/opt/flink$` 처럼 바뀌면 성공!)

### 명령어로 실행

```bash
flink run -py playground/src/(내가만든 py이름).py
```

- 환경 변수(PATH)가 잡혀있어서, 굳이  ./bin/flink 경로를 다 안 써도 됩니다.

### **결과 확인**

- `Job has been submitted...` 메시지 확인.
- 웹 대시보드(localhost:8081)에서 **Stdout** 로그 확인.

### 로그 확인

```bash
docker compose -f compose.session.yml logs -f taskmanager
```

