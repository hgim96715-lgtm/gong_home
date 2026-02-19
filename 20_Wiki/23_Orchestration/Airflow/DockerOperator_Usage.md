---
aliases:
  - DockeOperator
  - Airflow
tags:
  - Docker
  - Airflow
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_Installation]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Airflow_DAG_Operators]]"
---
## 개념 한 줄 요약

**"Airflow야, 너는 지시만 해. 실제 작업은 깨끗한 새 컨테이너에서 하고 버릴게."** 
(내 로컬 파이썬 환경을 더럽히지 않고, 태스크별로 완벽하게 격리된 환경을 쓰는 기술)

---
## 왜 필요한가? (The Problem: Dependency Hell) 🤯

Airflow Worker에는 수많은 라이브러리가 깔려 있습니다. 그런데 만약...

- **Task A:** `pandas==1.0.0`이 필요함 (레거시 코드)
- **Task B:** `pandas==2.0.0`이 필요함 (최신 기능 사용)

이 두 가지를 하나의 Airflow Worker 안에서 돌리려면 **버전 충돌**로 지옥이 펼쳐집니다.

**해결책 (DockerOperator):**
- Task A용 컨테이너를 하나 띄워서 실행하고 **삭제(Kill)**.
- Task B용 컨테이너를 하나 띄워서 실행하고 **삭제(Kill)**.
- Airflow는 그저 **"컨테이너 떴니? 끝났니?"** 감시만 합니다.

---
## 필수 준비물 (Prerequisites) 

이 기능을 쓰려면 Airflow가 내 컴퓨터의 도커를 조종할 수 있어야 합니다. (이미 [[Airflow_Installation]] 단계에서 설정했습니다!)

1. **Provider 패키지 설치:** (보통 기본으로 깔려있지만 없다면) `pip install apache-airflow-providers-docker`
2. **소켓 연결 확인:** `docker-compose.yaml`의 Airflow Worker 부분에 아래 줄이 반드시 있어야 합니다.

```yaml
# docker-compose.yaml - Airflow Worker에 반드시 있어야 함
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

```python
from airflow.providers.docker.operators.docker import DockerOperator
from docker.types import Mount # 파일 연결할 때 필요
```

## 전체 파라미터 해설

```python
task = DockerOperator(
    task_id='run_spark_job',           # ① 태스크 이름
    image='apache/spark:3.5.1',         # ② 어떤 컨테이너?
    api_version='auto',                # ③ 도커 버전 자동 감지
    command='python /app/daily_report.py',  # ④ 실행할 명령어
    auto_remove=True,                  # ⑤ 끝나면 컨테이너 삭제
    network_mode='project_default',    # ⑥ 네트워크 (DB 접속 핵심!)
    mounts=[                           # ⑦ 파일 공유
        Mount(source='/절대경로/apps', target='/app', type='bind')
    ],
    environment={                      # ⑧ 환경변수
        'DB_USER': 'airflow',
        'DB_PASSWORD': 'airflow',
    }
)
```

---
## 핵심 파라미터 4대장 (Deep Dive)

### ① `task_id` — 내가 원하는 이름으로 자유롭게

```python
task_id='run_spark_job'     # ✅ 아무 이름이나 OK
task_id='daily_report'      # ✅
task_id='abc'               # ✅
```

>DAG 안에서 **중복만 안 되면** 뭐든 상관없다. 단, 나중에 로그 볼 때 이름으로 찾으니까 **의미있게** 짓는 게 좋다.

### ② `image` — 파이썬? Spark? 언제 뭘 쓰나?

> 핵심 기준: "이 태스크가 뭘 실행하냐?"

| 상황             | 사용할 이미지              | 예시                        |
| -------------- | -------------------- | ------------------------- |
| 일반 파이썬 스크립트 실행 | `python:3.9-slim`    | 데이터 전처리, API 호출           |
| Spark 집계/분석 실행 | `apache/spark:3.5.1` | `spark-submit` 으로 실행하는 코드 |
| 커스텀 환경 필요      | 직접 만든 이미지            | 특수 라이브러리 필요할 때            |

```python
# 일반 파이썬 스크립트라면
image='python:3.9-slim'

# spark-submit으로 돌리는 코드라면, 내가설치한 Spark image
image='apache/spark:3.5.1'
```

> **Spark 버전 맞추기:** `docker-compose.yaml`에서 Spark 이미지 버전 확인 후 동일하게 쓸 것. 버전이 다르면 호환 문제가 생길 수 있다.

```bash
# 현재 띄워진 Spark 이미지 확인
docker ps --format "table {{.Image}}"
```

> 만약 나온 image이름이 apache/spark:3.5.1 이게아니라 다른거라면 그걸로 변경 더 자세히는` docker images` 확인 

### ③ `api_version` — 그냥 'auto'로 두면 된다, 도커 버전 자동감지 

```python
api_version='auto'  # ✅ 도커 버전 자동 감지 → 항상 이걸로
```

>직접 버전을 명시(`'1.41'` 등)할 수도 있지만, `'auto'`가 내 도커 버전에 맞게 알아서 맞춰주므로 신경 안 써도 된다.

### ④ `command` — 컨테이너 켜지자마자 실행할 명령어

```python
# 짧은 명령어
command='python /app/daily_report.py'

# 긴 명령어 (spark-submit)
command="""
    spark-submit \
    --master spark://spark-master:7077 \
    --jars /opt/spark/jars/postgresql-42.7.1.jar \
    /app/daily_report.py
"""
```

### ⑤ `auto_remove` — 항상 True

```python
auto_remove=True   # ✅ 끝나면 컨테이너 자동 삭제
auto_remove=False  # ❌ 쓰레기 컨테이너가 쌓임
```

### ⑥ `network_mode` — DB 접속의 핵심

컨테이너는 기본적으로 **격리된 공간**이다.
`network_mode`를 안 맞추면 컨테이너가 `postgres` 같은 다른 컨테이너를 **이름으로 못 찾는다.**

```scss
bridge (기본값)     → 인터넷은 되지만, postgres 컨테이너를 못 찾음
host               → 내 컴퓨터 네트워크 그대로 (맥에서는 잘 안 됨)
project_default    → ✅ docker-compose 네트워크 공유 → postgres:5432 접속 가능
```

**현재 사용 중인 네트워크 이름 확인**

```python
# 현재 사용 중인 네트워크 이름 확인
# 터미널에서:
docker network ls

# 보통 '프로젝트폴더명_default' 형태로 나온다
# 예: project_default, myproject_default

# 여기서 헷갈린다면
# 본인의 DB 컨테이너 이름 (예: project-postgres-1)
docker inspect project-postgres-1 -f '{{range $k, $v := .NetworkSettings.Networks}}{{println $k}}{{end}}'
```

```python
network_mode='project_default'  # ✅ docker network ls 로 확인한 이름
```

>**왜 이게 중요한가?** `daily_report.py` 안에서 `jdbc:postgresql://postgres:5432/airflow` 로 접속하는데, 같은 네트워크 안에 있어야 `postgres`라는 이름을 알아볼 수 있다. 
>네트워크가 다르면 `postgres`가 뭔지 몰라서 연결 자체가 안 된다.

### ⑦ `mounts` — 내 코드를 컨테이너에 빌려주기

컨테이너는 **빈 박스**다. 내 `daily_report.py` 파일이 컨테이너 안에 없으면 실행이 안 된다. 
`mounts`로 내 컴퓨터 폴더를 컨테이너 안에 연결(마운트)해준다.

```python
mounts=[
    Mount(
        source='/Users/gong/project/apps',  # 내 컴퓨터 경로 (절대경로 필수!)
        target='/app',                       # 컨테이너 안에서 보일 경로
        type='bind'                          # 항상 'bind'
    )
]
```

```scss
내 컴퓨터                          컨테이너 안
/Users/gong/project/apps/    →    /app/
    daily_report.py          →        daily_report.py  ← 이걸 command로 실행
```

>**절대경로 주의!**
> ❌ `./apps` (상대 경로 → 에러)
>  ✅ `/Users/gong/project/apps` (절대 경로)

### ⑧ mounts와 command의 관계

핵심: source/target/command는 각자 다른 세계의 주소

```text
[내 맥북]                              [새로 만든 컨테이너]

source =                   →→→        target =
/Users/gong/.../spark_app/            /opt/spark/apps/
      daily_report.py      →→→              daily_report.py
```

- **source** = 내 맥북에 있는 실제 경로 (맥북 입장에서 씀)
- **target** = 컨테이너 안에서 부를 가상 경로 (컨테이너 입장에서 씀)
- **command** = 컨테이너가 켜진 후 실행할 명령어 → **반드시 target 경로 기준으로 씀**

```python
# ✅ 컨테이너는 target 경로만 알아!
command="spark-submit /opt/spark/apps/daily_report.py"

# ❌ 컨테이너는 내 맥북 경로를 모름
command="spark-submit /Users/gong/.../daily_report.py"
```

### /opt/spark/apps 폴더, 누가 만드나?

`docker-compose.yml`에 이미 이렇게 되어 있는지 확인 

```yaml
# spark-master 컨테이너 설정
volumes: - ./spark_app:/opt/spark/apps ← spark-master가 뜰 때 이미 연결됨
```

즉 **spark-master 컨테이너가 뜨는 순간, `/opt/spark/apps`가 자동으로 생깁니다.** 도커가 target 경로를 자동 생성해줍니다. 직접 만들 필요 없습니다.


### ⑨ docker exec vs DockerOperator

- docker exec → 이미 떠있는 spark-master 안으로 들어가서 실행
- DockerOperator → 새 컨테이너를 생성해서 실행하고 삭제

```text
[docker exec 방식]

spark-master (항상 켜져있음)
    └─ /opt/spark/apps/daily_report.py  ← 파일이 이미 있음
         ↑
         여기 들어가서 실행


[DockerOperator 방식]

Airflow
  └─ 새 컨테이너 생성
        └─ mounts로 파일 연결 → 실행 → 컨테이너 삭제
```

|구분|docker exec|DockerOperator|
|---|---|---|
|컨테이너|spark-master (상주)|새로 생성 → 삭제|
|파일|이미 내부에 있음|mounts로 연결 필요|
|언제 씀|터미널 수동 테스트|Airflow 자동화 스케줄링|

---
## 트러블슈팅 (Troubleshooting) 🚑

### Q1. "DockerNotFound: Is the docker daemon running?"

- **원인:** Airflow가 도커를 못 찾겠대요.
- **해결:** `docker-compose.yaml`에서 `/var/run/docker.sock` 볼륨 연결을 빼먹었거나, 재시작을 안 했습니다.

### Q2. "FileNotFoundError" (컨테이너 안에서)

- **원인:** `mounts` 설정이 잘못되었습니다.
- **해결:** `Source` 경로가 내 컴퓨터에 실제로 존재하는지, `Target` 경로가 컨테이너 안에 잘 연결됐는지 확인하세요.

### Q3. "Connection Refused" (DB 접속 실패)

- **원인:** 컨테이너가 격리된 네트워크에 떠서 DB를 못 찾습니다.
- **해결:** `network_mode`를 `docker-compose` 네트워크 이름(예: `project_default`)으로 맞춰주세요.