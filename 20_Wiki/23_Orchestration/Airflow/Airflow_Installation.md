---
aliases:
  - Airflow 설치
  - Docker로 설치
  - Standalone 설치
  - 환경설정
tags:
  - Airflow
  - 설치
  - Docker
  - 환경설정
related:
  - "[[00_Airflow_Hompage]]"
  - "[[Common_Airflow_Errors]]"
---
## 개념 한 줄 요약

Airflow를 내 맥북에 설치하는 방법은 크게 **"파이썬 가상환경에 직접 깔기(Standalone)"** 와
**"컨테이너로 깔끔하게 띄우기(Docker)"** 두 가지가 있습니다.

[설치](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)

---
## 왜 필요한가 (Why)

**문제 상황:**
Airflow는 무거운 도구입니다. 맥북 로컬 파이썬에 그냥 깔아버리면 (`pip install apache-airflow`), 
나중에 다른 파이썬 프로젝트랑 **라이브러리 버전 충돌(Dependency Hell)** 이 나서 꼬일 확률이 99%입니다.

**해결책:**
* **학습용(초간단):** `Standalone` 모드로 빠르게 실행만 해봅니다.
* **실무/권장:** `Docker`를 사용해 내 컴퓨터 환경과 완전히 격리시킵니다. (나중에 지울 때도 컨테이너만 날리면 돼서 깔끔합니다.)

---
## 방법 1. Docker로 설치하기 (강력 추천 ⭐)

> 맥북에 'Docker Desktop'이 먼저 깔려 있어야 합니다.

### docker-compose 파일 다운로드

터미널을 열고 원하는 폴더로 가서 아래 명령어를 입력합니다. (공식 yaml 파일 다운로드)

```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/3.1.6/docker-compose.yaml'
```

### 폴더 생성 및 권한 설정

Airflow가 로그와 데이터를 저장할 폴더를 만듭니다.

```bash
mkdir -p ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

- **`id -u`**: 현재 맥북을 쓰고 있는 당신의 ID 번호를 알아냅니다. (보통 `501`입니다.)
- **`AIRFLOW_UID=...`**: Airflow에게 "너의 주인님 ID는 501번이야"라고 변수를 지정합니다.
- **`> .env`**: 이 내용을 `.env`라는 이름의 숨김 파일로 저장합니다.

### Airflow 실행

```bash
docker compose up -d
```

- `-d`: 백그라운드에서 실행하라는 뜻입니다.

### 상태 확인 및 접속

```bash
docker ps
```

- 위 명령어를 치면 `webserver`, `scheduler`, `worker` 등의 컨테이너가 모두 `healthy` 상태로 떠야 합니다.
- **접속 주소:** `localhost:8080`
- **기본 계정:** ID: `airflow` / PW: `airflow`

---
## 방법 2. Standalone 설치하기 (가벼운 테스트용)

> 파이썬만 있으면 바로 설치되지만, 환경이 꼬일 수 있으니 가상환경(venv) 안에서 하세요.

### 설치 및 환경변수 설정

```bash
# Airflow 홈 디렉토리 설정
export AIRFLOW_HOME=~/airflow

# 파이썬 버전에 맞는 제약조건 파일(Constraint) URL 생성
AIRFLOW_VERSION=2.6.2
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="[https://raw.githubusercontent.com/apache/airflow/constraints-$](https://raw.githubusercontent.com/apache/airflow/constraints-$){AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

# 설치 진행
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

### 실행

```bash
airflow standalone
```

- 실행하면 터미널에 `admin` 계정의 **임시 비밀번호**가 출력됩니다. 그걸 복사해서 로그인해야 합니다.

---
## 초보자가 자주 착각하는 포인트

1. **"도커 설치가 너무 복잡해요. 그냥 pip install 하면 안 되나요?"**
    - 당장은 편하지만, 나중에 DB 연결하고 Worker 늘리려고 할 때 10배 더 고생합니다. 
    - **맥북 사용자라면 무조건 Docker 방식을 추천합니다.**
        
2. **"localhost:8080이 안 들어가져요."**
    - Docker가 켜지는 데 시간이 좀 걸립니다(약 30초~1분). 
    - `docker ps` 명령어로 상태가 `healthy`가 될 때까지 기다리세요.
