---
aliases:
  - Airflow Operator
  - BashOperator
  - PythonOperator
  - 오퍼레이터 종류
  - Sensor
  - DockerOperator
tags:
  - Airflow
related:
  - "[[Airflow_Architecture]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_TaskFlow_API]]"
  - "[[Airflow_XComs]]"
---
# Airflow_Operators — 오퍼레이터 완벽 정리

## 한 줄 요약

```
DAG(공장) 안에서 실제 작업을 수행하는 작업자 템플릿
"뭘 실행할지" 를 정의하는 미리 만들어진 도구 상자
```

---

---

# ① 오퍼레이터 종류

|분류|역할|대표 예시|
|---|---|---|
|**Action**|실제 연산·명령 수행|`BashOperator`, `PythonOperator`, `DockerOperator`|
|**Transfer**|데이터를 A → B 로 이동|`S3FileTransferOperator`|
|**Sensor**|조건 만족까지 대기|`FileSensor`, `TimeSensor`|
|**Utility**|흐름 제어·표지판 역할|`EmptyOperator` (구 `DummyOperator`)|

```
실무 DAG 의 90%는 아래 3가지로 이루어짐:
  PythonOperator   내가 직접 파이썬으로 ETL 짤 때
  BashOperator     dbt, Spark 등 외부 명령어 실행할 때
  DockerOperator   라이브러리 버전 충돌 없이 격리 환경에서 돌릴 때
```

---

---

# ② BashOperator — 터미널 명령어 실행

```python
from airflow.operators.bash import BashOperator

task = BashOperator(
    task_id     = 'print_date',   # DAG 안에서 유일해야 함
    bash_command= 'date',         # 터미널에 치는 명령어 그대로
)
```

```python
# 여러 줄 명령어
task = BashOperator(
    task_id     = 'run_script',
    bash_command= """
        cd /opt/spark/apps &&
        python daily_report.py
    """,
)
```

---

---

# ③ PythonOperator — 파이썬 함수 실행

```python
from airflow.operators.python import PythonOperator

def my_function():
    print("Hello, Airflow!")

task = PythonOperator(
    task_id        = 'python_task',
    python_callable= my_function,    # 함수 이름만 (괄호 없음!)
)
```

```
⚠️ python_callable=my_function()  ← () 붙이면 DAG 로딩 시 즉시 실행됨
   python_callable=my_function    ← 함수 객체만 전달 (올바른 방법)
```

---

---

# ④ DockerOperator — 격리 컨테이너 실행 ⭐️

## 왜 필요한가?

```
Dependency Hell (의존성 지옥):
  Task A: pandas==1.0.0 필요 (레거시)
  Task B: pandas==2.0.0 필요 (최신)
  → 하나의 Worker 안에서 버전 충돌

DockerOperator 해결책:
  Task A용 컨테이너 실행 → 끝나면 삭제
  Task B용 컨테이너 실행 → 끝나면 삭제
  Airflow 는 "컨테이너 떴니? 끝났니?" 감시만
```

## 사전 준비

```yaml
# docker-compose.yml — Airflow Worker 에 반드시 추가
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
  # Airflow 가 내 도커를 조종할 수 있도록 소켓 연결
```

```python
from airflow.providers.docker.operators.docker import DockerOperator
from docker.types import Mount
```

## 전체 파라미터

```python
task = DockerOperator(
    task_id     = 'run_spark_job',
    image       = 'apache/spark:3.5.1',
    api_version = 'auto',
    command     = 'python /app/daily_report.py',
    auto_remove = True,
    network_mode= 'train-network',
    mounts      = [
        Mount(
            source= '/Users/gong/project/spark_app',  # 내 컴퓨터 경로 (절대경로!)
            target= '/app',                            # 컨테이너 안 경로
            type  = 'bind'
        )
    ],
    environment = {
        'DB_USER'    : 'train_user',
        'DB_PASSWORD': 'train_password',
    },
)
```

## 핵심 파라미터 상세

### image — 뭘 실행하냐에 따라 결정

|상황|이미지|
|---|---|
|일반 파이썬 스크립트|`python:3.9-slim`|
|Spark 집계·분석|`apache/spark:3.5.1`|
|커스텀 환경|직접 빌드한 이미지|

```bash
# 현재 띄워진 Spark 이미지 버전 확인 (docker-compose 와 맞춰야 함)
docker ps --format "table {{.Image}}"
docker images   # 더 자세히
```

### api_version — 항상 auto

```python
api_version='auto'   # 도커 버전 자동 감지 → 항상 이걸로
```

### auto_remove — 항상 True

```python
auto_remove=True    # ✅ 끝나면 컨테이너 자동 삭제
auto_remove=False   # ❌ 쓰레기 컨테이너가 서버에 쌓임
```

### network_mode — DB 접속의 핵심

```
컨테이너는 기본적으로 격리된 공간
→ network_mode 를 안 맞추면 postgres 같은 컨테이너를 이름으로 못 찾음

bridge (기본값)   인터넷은 되지만 postgres 컨테이너 못 찾음
host              내 컴퓨터 네트워크 그대로 (맥에서는 잘 안 됨)
train-network     ✅ docker-compose 네트워크 공유 → postgres:5432 접속 가능
```

```bash
# 현재 사용 중인 네트워크 이름 확인
docker network ls
# 보통 '프로젝트폴더명_default' 또는 직접 지정한 이름

# 특정 컨테이너의 네트워크 확인
docker inspect train-postgres -f '{{range $k, $v := .NetworkSettings.Networks}}{{println $k}}{{end}}'
```

### mounts — 내 코드를 컨테이너에 연결

```
컨테이너는 빈 박스
→ daily_report.py 가 컨테이너 안에 없으면 실행 불가
→ mounts 로 내 폴더를 컨테이너 안에 연결
```

```python
mounts=[
    Mount(
        source= '/Users/gong/project/spark_app',  # 내 맥북 경로 (절대경로 필수!)
        target= '/app',                            # 컨테이너 안에서 보이는 경로
        type  = 'bind'
    )
]
```

```
내 맥북                               컨테이너 안
/Users/gong/project/spark_app/  →   /app/
    daily_report.py              →       daily_report.py

⚠️ source 는 절대경로 필수
   ❌ ./spark_app     (상대경로 → 에러)
   ✅ /Users/gong/project/spark_app
```

### command 는 target 경로 기준

```python
# ✅ 컨테이너는 target 경로(/app) 만 알고 있음
command='python /app/daily_report.py'

# ❌ 컨테이너는 내 맥북 경로를 모름
command='python /Users/gong/project/spark_app/daily_report.py'
```

```
source = 내 맥북에서 쓰는 경로
target = 컨테이너 안에서 쓰는 경로
command = target 경로 기준으로 작성
```

## docker exec vs DockerOperator

|구분|docker exec|DockerOperator|
|---|---|---|
|컨테이너|이미 떠있는 컨테이너 진입|새로 생성 → 실행 → 삭제|
|파일|이미 내부에 있음|mounts 로 연결 필요|
|용도|터미널 수동 테스트|Airflow 자동화 스케줄링|

---

---

# ⑤ EmptyOperator — 표지판·흐름 제어

```
구 DummyOperator → 현재 공식 명칭 EmptyOperator
실무에서는 여전히 DummyOperator 로 많이 부름
("바보(Dummy)" 어감이 좋지 않아 "비어있음(Empty)" 으로 변경)
```

## 용도 1 — Start / End 표지판

```python
from airflow.operators.empty import EmptyOperator

start = EmptyOperator(task_id='start')
end   = EmptyOperator(task_id='end')

start >> [task1, task2] >> end
```

## 용도 2 — 의존성 모으기 (Fan-in / Fan-out)

```
A, B, C 가 다 끝나면 D, E, F 를 실행할 때

EmptyOperator 없이:
  A>>D, A>>E, A>>F, B>>D, B>>E, B>>F, C>>D, C>>E, C>>F  (9개 선)

EmptyOperator 있으면:
  [A, B, C] >> join >> [D, E, F]  (깔끔)
```

```python
join = EmptyOperator(task_id='join')

[task_a, task_b, task_c] >> join >> [task_d, task_e, task_f]
```

---

---

# ⑥ Sensor — 조건 만족까지 대기

```
Sensor 도 Operator 의 일종
조건이 만족될 때까지 계속 실행하면서 확인

"파일 왔어?" (아니) → 30초 대기 → "파일 왔어?" (아니) → ...

⚠️ Sensor 를 잘못 쓰면 Worker 슬롯을 독점
   다른 작업이 못 돌아가는 Deadlock 발생 가능
```

```python
from airflow.sensors.filesystem import FileSensor

wait_for_file = FileSensor(
    task_id       = 'wait_for_data',
    filepath      = '/data/input.csv',
    poke_interval = 30,    # 30초마다 확인
    timeout       = 3600,  # 1시간 후 타임아웃
)
```

---

---

# ⑦ 의존성 설정 — >> 연산자

```python
# 순차 실행
task1 >> task2 >> task3

# 병렬 실행
start >> [task1, task2] >> end

# 여러 선 연결
task1 >> task3
task2 >> task3   # task1, task2 가 둘 다 끝나야 task3 실행
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`DockerNotFound`|docker.sock 볼륨 연결 누락|`docker-compose.yml` 에 `/var/run/docker.sock` 추가|
|`FileNotFoundError` (컨테이너 안)|mounts source 경로 오류|절대경로 확인, 실제 파일 존재 여부 확인|
|`Connection Refused` (DB)|network_mode 불일치|`docker network ls` 로 네트워크 이름 확인 후 수정|
|`python_callable` 즉시 실행|함수에 `()` 붙임|`python_callable=func` (괄호 제거)|
|`task_id` 중복 에러|같은 DAG 안에 동일 ID|DAG 안에서 task_id 유일하게 설정|
|Sensor Deadlock|Worker 슬롯 독점|`poke_interval` / `timeout` 설정 or `mode='reschedule'` 사용|