---
aliases:
  - Airflow 파이프라인
  - DAG
  - 배치 스케줄링
  - train_schedule_dag
  - "train_delay_dagtags:"
tags:
  - Project
related:
  - "[[03_Kafka_Producer]]"
  - "[[04_Spark_Streaming]]"
  - "[[05_Superset_Dashboard]]"
  - "[[PostgreSQL_Setup]]"
  - "[[Linux_OpenSSL]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[Airflow_TaskFlow_API]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Python_Sys_Module]]"
  - "[[Airflow_Execution_Date]]"
  - "[[00_Seoul_Station_Real-Time_Train_Project|train-projcet]]"
---

# 06_Airflow_Pipeline — 배치 스케줄링

## 이 단계의 목표

```
Producer 의 while 루프 안에서 시간 체크로 처리하던
"하루 1회 배치 작업" 을 Airflow DAG 으로 분리

기존:
  while 루프 안에서
  → 60초 폴링 + 01:00~03:00 시간 체크 + 운행계획(하루1회) 전부 관리

변경:
  while 루프    → 60초 realtime 폴링만
  운행계획 수집 → train_schedule_dag (매일 00:05)
  지연 분석     → train_delay_dag    (매일 02:00)
```

---

---

# ① Docker Compose — Airflow 서비스 추가

## docker-compose.yml

```yaml
  airflow:
    image: apache/airflow:2.9.1
    container_name: train-airflow
    env_file:
      - .env
    depends_on:
      - postgres
      - kafka
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
      AIRFLOW__CORE__FERNET_KEY: ''
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
      AIRFLOW__WEBSERVER__SECRET_KEY: ${AIRFLOW_SECRET_KEY}
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    volumes:
      - ./airflow/dags:/opt/airflow/dags
      - ./producer:/opt/airflow/producer   # producer 코드 공유
    ports:
      - "8082:8080"   # Airflow Web UI (8080=Spark, 8088=Superset 충돌 방지)
    networks:
      - train-network
	command: >
	  bash -c "
	    pip3 install kafka-python requests &&
	    airflow db migrate &&
	    airflow users create \
	      --username admin \
	      --password admin \
	      --firstname Admin \
	      --lastname User \
	      --role Admin \
	      --email admin@example.com || true &&
	    airflow scheduler &
	    airflow webserver
	  "
```

```
⚠️ command 의 \ (역슬래시) 주의
  Bash 에서 명령어를 다음 줄로 이어가려면 끝에 \ 필수
  없으면 airflow users create 까지만 읽고 에러 → 계정 생성 실패
```

## postgres/init.sql — airflow DB 추가

```sql
-- init.sql 맨 위에 추가
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```

```bash
# 볼륨이 이미 있어서 init.sql 이 재실행 안 될 경우
# psql 직접 접속해서 수동 실행
docker exec -it train-postgres psql -U train_user -d train_db

CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
\q
```

> [[PostgreSQL_Setup#⑥ 스키마 수정 — down -v 없이 변경하기 ⭐️|⑥ 스키마 수정]]  참고

## .env 추가

```bash
AIRFLOW_SECRET_KEY=your_secret_key_here
# openssl rand -base64 42 로 생성
```

> [[Linux_OpenSSL]] 참고

---

---

# ② 폴더 구조

```
seoul-train-realtime-project/
├── airflow/
│   └── dags/
│       ├── train_schedule_dag.py   ← 매일 00:05 운행계획 수집
│       └── train_delay_dag.py      ← 매일 02:00 지연 분석
└── producer/
    ├── main.py
    └── producer.py                 ← realtime 폴링만 담당
```

---

---

# ③ DAG 1 — 운행계획 수집 (매일 00:05)

```
파일: airflow/dags/train_schedule_dag.py
역할: 매일 00:05 당일 운행계획 API → train-schedule 토픽 발행
      producer.py 의 run_schedule() 재사용
```

```python
# airflow/dags/train_schedule_dag.py
import sys
import pendulum
from airflow.decorators import dag, task

default_args = {
    'owner'          : 'airflow',
    'retries'        : 1,
    'retry_delay'    : pendulum.duration(minutes=5),
    'depends_on_past': False,
    'email_on_failure': False,
}

@dag(
    dag_id      = 'train_schedule_dag',
    default_args= default_args,
    description = '매일 00:05 → 당일 서울역 출발 열차 운행계획 수집',
    schedule    = '5 0 * * *',
    start_date  = pendulum.datetime(2026, 3, 15, tz='Asia/Seoul'),
    catchup     = False,
    tags        = ['train', 'schedule'],
)
def train_schedule_pipeline():

    @task
    def collect_schedule(**context):
        sys.path.insert(0, '/opt/airflow/producer')
        from producer import TrainProducer

        run_ymd = pendulum.now('Asia/Seoul').strftime('%Y%m%d')
        print(f"[운행계획 DAG] 수집 시작 🚆 - {run_ymd}")

        tp = TrainProducer()
        tp.run_schedule(run_ymd)
        tp.producer.flush()
        tp.producer.close()
        print(f"[운행계획 DAG] 수집 완료 ✅ - {run_ymd}")

    collect_schedule()

train_schedule_pipeline()
```

```
run_ymd 에 pendulum.now() 를 쓰는 이유:
  @daily 스케줄에서 logical_date = 어제
  운행계획은 "오늘" 데이터가 필요 → pendulum.now() 로 실제 오늘 사용
```

---

---

# ④ DAG 2 — 지연 분석 (매일 02:00)

```
파일: airflow/dags/train_delay_dag.py
역할: 매일 02:00 전날 계획 vs 실제 비교 → train-delay 토픽 발행
      producer.py 의 run_delay_analysis() 재사용
```

```python
# airflow/dags/train_delay_dag.py
import sys
import pendulum
from airflow.decorators import dag, task

default_args = {
    'owner'          : 'airflow',
    'retries'        : 2,                          # 지연분석은 API 호출 많으므로 재시도 2회
    'retry_delay'    : pendulum.duration(minutes=10),
    'depends_on_past': False,
    'email_on_failure': False,
}

@dag(
    dag_id      = 'train_delay_dag',
    default_args= default_args,
    description = '매일 새벽 02:00 → 전날 서울역 출발 열차 지연 정보 수집',
    schedule    = '0 2 * * *',
    start_date  = pendulum.datetime(2026, 3, 14, tz='Asia/Seoul'),
    catchup     = False,
    tags        = ['train', 'delay'],
)
def train_delay_pipeline():

    @task
    def run_delay_analysis_task(logical_date=None, **context):
        # logical_date: Airflow 가 자동 주입하는 pendulum 객체 (= 어제 날짜)
        sys.path.insert(0, '/opt/airflow/producer')
        from producer import TrainProducer

        kst_date = logical_date.in_timezone('Asia/Seoul')

        # ⚠️ Actions (수동 Trigger) 실행 시 오늘 날짜로 들어오는 문제 방어
        # 스케줄 자동 실행(새벽 2시) 시에는 logical_date = 어제 → 주석 유지
        # 수동 실행인데 어제 날짜로 분석하고 싶을 때만 주석 해제
        # now_kst = pendulum.now('Asia/Seoul')
        # if kst_date.date() == now_kst.date():
        #     kst_date = kst_date.subtract(days=1)

        target_date = kst_date.strftime('%Y%m%d')
        print(f"[지연 정보 DAG] 수집 시작 🚆 - {target_date} 기준 열차 지연 정보")

        tp = TrainProducer()
        tp.run_delay_analysis(target_date)
        tp.producer.flush()
        tp.producer.close()
        print(f"[지연 정보 DAG] 수집 완료 ✅ - {target_date} 기준 열차 지연 정보")

    run_delay_analysis_task()

train_delay_pipeline()

```

```
target_date 에 logical_date 를 쓰는 이유:
  @daily 스케줄에서 logical_date = 어제
  지연분석은 "어제" 데이터가 필요 → logical_date 그대로 사용
  subtract(days=1) 하면 그저께가 되므로 주의

⚠️ Actions (수동 Trigger) 버튼 누르면 오늘 날짜가 들어옴:
  Web UI → DAG → ▶ Actions → Trigger DAG
  → logical_date = 지금 이 순간 (오늘)
  → target_date = 오늘 날짜로 분석됨

  자동 실행 (새벽 02:00):
    logical_date = 어제 → 정상 ✅

  수동으로 어제 날짜로 테스트하고 싶을 때:
  방법 1: Trigger DAG w/ config 에서 날짜 직접 지정
          {"logical_date": "2026-03-15T00:00:00+09:00"}
  방법 2: 위 주석 코드 해제 (오늘이면 하루 빼기)
```

> [[Airflow_Execution_Date]] 참고

---

---

# ⑤ producer.py 수정 — run_delay_analysis 파라미터 추가

```
기존: 함수 내부에 datetime.now() 고정 → 항상 "지금 이 순간의 어제"만 분석
      Airflow 가 과거 날짜로 재실행(Backfill) 해도 무시됨

변경: 날짜를 파라미터로 받아서 Airflow 가 원하는 날짜 처리 가능
      파라미터 없으면 기존처럼 어제로 동작 (while 루프 호환)
```

```python
def run_delay_analysis(self, target_date: Optional[str] = None):

    # 파라미터로 날짜 안 줬으면 기존처럼 어제로 계산 (while 루프 호환)
    if target_date is None:
        target_date = (datetime.now() - timedelta(days=1)).strftime("%Y%m%d")

    print(f"\n[지연분석] {target_date} 분석 시작!")

    if self.state.get("last_delay_analysis_date") == target_date:
        print(f"[지연분석] 이미 {target_date} 분석 기록 있음 → 스킵")
        self.delay_done_today = True
        return

    plan_items = self.train_info.get_train_schedule(target_date)
    # 이하 코드에서 yesterday → target_date 로 전부 변경
    # actual_items = self.train_info.get_train_realtime(target_date, trn_no)
    # self._send("train-delay", {"run_ymd": target_date, ...})
    # self.state["last_delay_analysis_date"] = target_date
```

---

---

# ⑥ Airflow Web UI 접속

```
http://localhost:8082
ID: admin / PW: admin
```

## DAG 활성화 및 실행

```
⚠️ Airflow DAG 실행 전 아래가 먼저 실행되어 있어야 함
   → [[04_Spark_Streaming]] ⑥ 실행 참고
   ① docker compose up -d
   ② jar 파일 복사 + Kafka 토픽 생성
   ③ Producer 실행 (python3 producer.py)
   ④ Spark Consumer 실행 (spark-submit)
```

>[[04_Spark_Streaming#⑥ 실행]] 

```
1. Web UI → DAGs 탭
   train_schedule_dag → 토글 ON
   train_delay_dag    → 토글 ON

2. 수동 실행 (테스트)
   DAG 클릭 → ▶ Trigger DAG → Trigger

3. 실행 로그 확인
   DAG 클릭 → Grid View → Task 클릭 → Logs
```

## ⚠️ 버튼 눌러도 실행 안 될 때 — 포그라운드 스케줄러

```
Web UI 에서 Trigger 버튼을 눌렀는데 실행이 안 되는 경우:
  → 컨테이너 안의 스케줄러가 제대로 안 떠 있는 것

자주 발생하는 원인:
  docker compose up 을 처음 실행했을 때는 됐는데
  두 번째 재시작부터 갑자기 안 됨

  원인: command 의 && 체인 문제
    airflow users create ... &&   ← 이미 admin 계정 있으면 실패
    airflow scheduler &           ← && 에 막혀서 실행 안 됨!

  해결: users create 뒤에 || true 추가
    airflow users create ... || true &&
    airflow scheduler &           ← 이제 무조건 실행됨
```

```bash
# || true 추가 후 docker compose 재시작
docker compose down
docker compose up -d
```

### 그 다음에도 안될때 시도 

```bash
docker exec -it train-airflow airflow scheduler
```

```
왜 이렇게 해야 하나:

  docker-compose 의 command 에서 scheduler 와 webserver 를
  동시에 실행하고 있는데, 간혹 스케줄러가 백그라운드에서
  조용히 죽어있는 경우가 있음

  포그라운드로 직접 실행하면:
    ① 스케줄러가 살아있는지 바로 확인 가능
    ② 실행 로그가 터미널에 그대로 찍혀서 context 정보 확인 가능
    ③ 트리거 버튼을 눌렀을 때 즉시 반응하는지 확인 가능

  정상 실행 시 이런 로그가 찍힘:
```

```text
INFO - DagRun Finished: dag_id=train_schedule_dag,
  execution_date=2026-03-15T09:40:50+00:00,
  state=success,
  data_interval_start=2026-03-13T15:05:00+00:00,
  data_interval_end=2026-03-14T15:05:00+00:00
```

```
테스트 후에는 Ctrl+C 로 중단
실제 운영 시에는 docker-compose 가 알아서 백그라운드로 관리
```

---

---

# ⑦ 실행 결과 로그

## train_schedule_dag 성공 로그

```text
INFO - [운행계획]-> 818건 발행완료!
INFO - [운행계획 DAG] 수집 완료 ✅ - 20260315
INFO - Done. Returned value was: None
```

## train_delay_dag 성공 로그

```text
INFO - [지연분석] 142건 발행완료
INFO - [지연 정보 DAG] 수집 완료 ✅ - 20260314 기준 열차 지연 정보
INFO - Done. Returned value was: None
```

```
⚠️ 지연 분석 건수가 05_Superset 의 숫자와 다를 수 있음
   WHERE 조건 차이 때문 — 직접 확인해서 맞는 것 검증 완료
```

---

---

# ⑧ Producer 역할 변경

```
Airflow 도입 후 while 루프 단순화
배치 작업(운행계획, 지연 분석) → DAG 이 담당
while 루프 → 60초 realtime 폴링만
```

```python
def run(self):
    print(f"Train Producer [실시간 폴링 전용] 시작 (간격: {POLL_INTERVAL}초)\n")

    self.current_date = datetime.now().strftime("%Y%m%d")
    self.run_schedule(self.current_date)   # 폴링 대상 목록 로드용

    # ❌ 삭제: run_delay_analysis() 호출 (Airflow 담당)

    while True:
        now       = datetime.now()
        today_str = now.strftime("%Y%m%d")

        if today_str != self.current_date:
            print(f"\n날짜 변경 [{self.current_date} → {today_str}]")
            self.current_date = today_str
            self.run_schedule(self.current_date)

        # ❌ 삭제: 01:00~03:00 시간 체크 지연 분석 (Airflow 담당)

        # 실시간 폴링만 남음
        ...
        time.sleep(POLL_INTERVAL)
```

---

---
# ⑨ 이메일 알림 설정

```
새벽 2시까지 기다릴 수 없으니
DAG 성공/실패 시 Gmail 로 알림 받기
```

## Gmail 앱 비밀번호 발급

```
① Google 계정 → 보안 → 2단계 인증 ON (필수)
② 앱 비밀번호 → 앱: 메일 → 기기: 기타(직접 입력) → 생성
③ 16자리 비밀번호 복사 → .env 의 AIRFLOW_SMTP_PASSWORD 에 붙여넣기
   (띄어쓰기 없이: abcdefghijklmnop)
```

## .env 추가

```bash
AIRFLOW_SMTP_USER=본인@gmail.com
AIRFLOW_SMTP_PASSWORD=abcdefghijklmnop   # Gmail 앱 비밀번호 16자리 (띄어쓰기 제거)
```

## docker-compose.yml airflow 서비스에 추가

```yaml
environment:
  # ... 기존 설정 ...
  AIRFLOW__SMTP__SMTP_HOST: 'smtp.gmail.com'
  AIRFLOW__SMTP__SMTP_STARTTLS: 'True'
  AIRFLOW__SMTP__SMTP_SSL: 'False'
  AIRFLOW__SMTP__SMTP_PORT: '587'
  AIRFLOW__SMTP__SMTP_USER: ${AIRFLOW_SMTP_USER}
  AIRFLOW__SMTP__SMTP_PASSWORD: ${AIRFLOW_SMTP_PASSWORD}
  AIRFLOW__SMTP__SMTP_MAIL_FROM: ${AIRFLOW_SMTP_USER}   # 보내는 사람 = 본인 이메일
```

```
⚠️ x-airflow-common 패턴 아님
   우리 docker-compose 는 airflow 서비스 하나에 직접 추가
   x-airflow-common 은 Airflow 공식 템플릿 방식 (다른 구조)
```

## DAG default_args 에 이메일 설정

```python
default_args = {
    'owner'          : 'airflow',
    'retries'        : 2,
    'retry_delay'    : pendulum.duration(minutes=10),
    'email'          : ['본인@gmail.com'],   # 받을 이메일
    'email_on_failure': True,                # 실패 시 메일
    'email_on_retry' : False,                # 재시도마다 메일 (폭탄 방지 False)
}
```

```
email_on_failure=True  → 재시도 다 소진하고 최종 실패할 때 1번 메일
email_on_retry=False   → 재시도마다 메일 보내면 폭탄됨 → False 권장

적용 순서:
  docker compose down && docker compose up -d
  → 컨테이너 재시작해야 환경변수 반영됨
```

## 트러블슈팅

|증상|원인|해결|
|---|---|---|
|메일이 안 와요|앱 비밀번호 아닌 계정 비밀번호 사용|Gmail 앱 비밀번호 발급 후 사용|
|Connection refused|2단계 인증 미설정|Google 계정 2단계 인증 ON 필수|
|환경변수 적용 안 됨|컨테이너 재시작 안 함|`docker compose down && up -d`|


----
---

# ⑩ 항상 켜져있어야 하는 것

```
Airflow DAG 이 정상 동작하려면
아래 세 가지가 동시에 실행 중이어야 함
```

|프로세스|명령어|역할|
|---|---|---|
|Docker 컨테이너|`docker compose up -d`|Kafka / PostgreSQL / Spark / Airflow / Superset|
|Producer|`python3 producer.py`|60초마다 train-realtime 토픽 발행|
|Spark Consumer|`spark-submit ...`|Kafka → PostgreSQL 실시간 저장|

```
각각 끄면 어떻게 되나:

Producer 끄면
  → train-realtime 토픽에 새 데이터 없음
  → Superset 전광판 갱신 안 됨

spark-submit 끄면
  → Kafka 에 데이터는 쌓이지만 PostgreSQL 에 저장 안 됨
  → Superset 에 아무것도 안 보임

Airflow만 켜져있으면
  → 배치(운행계획, 지연분석) 만 동작
  → 실시간 전광판 동작 안 됨

맥북을 끄면
  → 전부 멈춤 (로컬 개발 환경)
```

```
Airflow DAG 이 하는 것:
  train_schedule_dag  → 매일 00:05 운행계획 수집 (배치)
  train_delay_dag     → 매일 02:00 지연 분석 (배치)
  → 이 두 가지는 Producer while 루프 에서 분리됨

Producer 가 하는 것:
  60초마다 train-realtime 토픽 발행 (실시간)
  → Airflow 가 대신 못 함
```

> 실행 명령어 전체 → [[04_Spark_Streaming]] ⑥ 실행 참고




---
---
# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`ModuleNotFoundError: producer`|sys.path 에 producer 경로 없음|`sys.path.insert(0, '/opt/airflow/producer')` 추가|
|airflow DB 연결 실패|airflow DB 미생성|init.sql 에 `CREATE DATABASE airflow` 추가 후 psql 수동 실행|
|DAG 가 목록에 안 보임|DAG 파일 파싱 에러|Web UI → DAGs → Import Errors 확인|
|Trigger 눌러도 실행 안 됨|스케줄러가 백그라운드에서 죽어있음|`docker exec -it train-airflow airflow scheduler` 포그라운드 실행|
|**처음 실행 후 재시작 시 DAG 안 뜸**|**`airflow users create` 가 이미 존재해서 실패 → `&&` 체인 끊김 → scheduler 실행 안 됨**|**`users create` 뒤에 `\| true` 추가**|
|Task 실패 후 재시도 안 됨|`retries: 0`|`default_args` 에 `retries: 1` 이상 설정|
|`catchup=True` 로 과거 실행 쌓임|기본값이 True|`catchup=False` 명시|
|`producer_state.json` 으로 스킵됨|이미 발행 기록 있음|파일 삭제 후 재실행|
|**지연분석 성공인데 0건 발행됨**|**공공데이터 API 에 전날 데이터가 새벽 2시에 아직 없음**|**스케줄을 오전 6~9시로 늦추기 + `delay_count==0` 이면 WARNING 로그**|
|`ModuleNotFoundError: No module named 'kafka'`|Airflow 컨테이너에 kafka-python 미설치|docker-compose.yml command 에 pip install 추가 (아래 참고)|
|`producer_state.json` 날짜가 안 바뀜|상대경로라 컨테이너마다 다른 위치에 저장됨|`state_file` 을 `os.path.abspath(__file__)` 기준 절대경로로 변경|

## ModuleNotFoundError: No module named 'kafka' 해결

```
원인:
  Airflow 컨테이너는 airflow 만 설치된 순수 환경
  producer.py 가 import 하는 kafka-python 이 없음
  → pip install 을 컨테이너 시작 시 추가해야 함
```

```yaml
# docker-compose.yml airflow command 수정
command: >
  bash -c "
    pip3 install kafka-python requests &&
    airflow db migrate &&
    airflow users create \
      --username admin \
      --password admin \
      --firstname Admin \
      --lastname User \
      --role Admin \
      --email admin@example.com &&
    airflow scheduler &
    airflow webserver
  "
```

```
pip install kafka-python requests
  kafka-python  → from kafka import KafkaProducer
  requests      → producer.py 의 API 호출용

컨테이너 재시작 시마다 pip install 이 실행됨
→ 느리지만 가장 간단한 방법

더 깔끔한 방법:
  requirements.txt 만들고 pip install -r /opt/airflow/producer/requirements.txt
  → producer/requirements.txt 에 이미 kafka-python, requests 있으면 재사용 가능
```

```bash
# requirements.txt 활용 시
command: >
  bash -c "
    pip install -r /opt/airflow/producer/requirements.txt &&
    airflow db migrate &&
    ...
  "
```

```
이메일 알림 성공 여부:
  에러가 났지만 이메일은 정상 전송됨 ✅
  Sent an alert email to ['본인@gmail.com']
  → SMTP 설정은 제대로 된 것
```


```text
Try 3 out of 3
  → retries=2 설정대로 3번 시도 후 최종 실패한 것

이메일에 있는 링크 두 가지:
  Log          → Airflow Web UI 로그 바로 이동
  Mark success → 실패한 태스크를 성공으로 강제 처리
                 (임시방편 — 근본 원인은 kafka-python 설치)
```


##  ⚠️ 지연분석 0건 — 공공데이터 API 등록 지연

```
증상:
  DAG 성공 표시 / 로그에 "정상" / 하지만 [지연분석] 0건 발행완료

원인:
  열차 실제 운행정보(실제 출발·도착 시각) 는
  코레일이 데이터 검증 후 API 에 등록
  → 자정 이후 바로 올라오지 않음
  → 오전 중에 올라오는 경우가 많음

  새벽 02:00 에 전날 데이터 조회
  → API 서버에 아직 없음 → 응답은 정상이지만 빈 데이터 → 0건

  Airflow 가 성공으로 표시하는 이유:
  코드가 예외(Exception) 없이 정상 종료됨
  0건이어도 에러를 던지지 않음 → Airflow 는 성공으로 판단
```

```
해결:
  스케줄을 오전 6~9시로 늦추기
  → API 에 데이터가 올라올 시간 충분히 확보

  테스트 방법:
  수동 Trigger 로 지금 당장 실행해서
  어제 날짜 데이터가 오는지 확인
  → 데이터 오면 그 시간대로 스케줄 변경
```


```python
# train_delay_dag.py 스케줄 변경
schedule = '0 2 * * *'    # ❌ 새벽 2시 → 데이터 없을 수 있음
schedule = '0 6 * * *'    #  오전 6시 변경 
```



---

✅ 완료되면 → [[07_Integration_Test]] 으로 이동