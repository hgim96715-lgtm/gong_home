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
  - "[[00_Airflow_HomePage]]"
  - "[[Docker_Compose_Setup]]"
  - "[[Airflow_Configuration]]"
---
# Airflow_Installation — Airflow 설치

## 한 줄 요약

```
학습용:  Standalone (pip install, 빠르게 실행)
실무용:  Docker Compose (격리 환경, 나중에 깔끔하게 제거 가능)
→ Docker 방식 강력 권장
```

---

---

# ① 왜 Docker 로 설치해야 하는가

```
pip install apache-airflow 로 맥북에 직접 설치하면:
  다른 프로젝트 라이브러리와 버전 충돌 (Dependency Hell)
  나중에 지우기도 복잡
  Worker 늘리거나 DB 연결할 때 10배 고생

Docker 방식:
  컨테이너로 완전 격리 → 내 컴퓨터 환경에 영향 없음
  지울 때 컨테이너만 삭제하면 끝
  Worker / DB / Scheduler 모두 한 번에 관리
```

---

---

# ② Docker Compose 설치 — 실무 권장

## 사전 조건

```
Docker Desktop 설치 필요
```

## Step 1 — 공식 docker-compose.yaml 다운로드

```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/3.1.6/docker-compose.yaml'
```

## Step 2 — 폴더 생성 + 권한 설정

```bash
mkdir -p ./dags ./logs ./plugins
echo -e "AIRFLOW_UID=$(id -u)" > .env
```

```
AIRFLOW_UID=$(id -u):
  현재 사용자 ID 번호 (보통 501)
  Airflow 컨테이너가 파일을 쓸 때 권한 충돌 방지
  .env 파일에 저장 → docker-compose.yaml 이 자동으로 읽음
```

## Step 3 — (선택) DockerOperator 사용 시 추가 설정

```
Airflow 에서 Docker 컨테이너를 직접 실행하려면
docker.sock 을 공유해야 함
```

```yaml
# docker-compose.yaml
services:
  airflow-worker:
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
      - /var/run/docker.sock:/var/run/docker.sock   # ← 추가
```

## Step 4 — 실행

```bash
docker compose up -d
```

## Step 5 — 상태 확인 + 접속

```bash
docker ps
# webserver / scheduler / worker 모두 healthy 상태 확인
```

```
접속:  http://localhost:8080
ID:    airflow
PW:    airflow
```

```
⚠️ localhost:8080 안 들어가질 때:
  컨테이너 뜨는 데 30초~1분 걸림
  docker ps 에서 healthy 확인 후 접속
```

---

---

# ③ Standalone 설치 — 학습용

```
가상환경(venv) 안에서 설치 권장
맥북 로컬에 직접 설치 → 다른 프로젝트에 영향 줄 수 있음
```

```bash
# 가상환경 먼저
python3 -m venv .venv
source .venv/bin/activate

# Airflow 홈 설정
export AIRFLOW_HOME=~/airflow

# 버전에 맞는 Constraint 파일로 설치
AIRFLOW_VERSION=2.6.2
PYTHON_VERSION="$(python --version | cut -d " " -f 2 | cut -d "." -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

```bash
# 실행
airflow standalone
```

```
실행하면 터미널에 admin 임시 비밀번호 출력
복사해서 로그인

접속: http://localhost:8080
```

---

---

# 방법 비교

|항목|Docker Compose|Standalone|
|---|---|---|
|환경 격리|✅ 완전 격리|❌ 로컬 환경에 설치|
|삭제|컨테이너 삭제로 끝|pip uninstall + 파일 정리|
|Worker / DB|자동 구성|별도 설정 필요|
|실행 속도|느림 (컨테이너)|빠름|
|권장 상황|실무 / 프로젝트|빠른 테스트|