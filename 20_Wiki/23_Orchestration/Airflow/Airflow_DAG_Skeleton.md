---
aliases:
  - DAG 템플릿
  - default_args
  - Pendulum
  - start_date
  - catchup
  - depends_on_past
tags:
  - Airflow
  - DAG
related:
  - "[[Airflow_DAG_Concept]]"
  - "[[Airflow_Operators]]"
  - "[[Airflow_UI_Usage]]"
  - "[[Airflow_TaskGroup]]"
  - "[[Catchup_and_Backfill]]"
  - "[[Terminal_Editors]]"
  - "[[Linux_Scheduling_Crontab]]"
  - "[[Airflow_Execution_Date]]"
---
# Airflow_DAG_Skeleton — DAG 기본 템플릿

## 한 줄 요약

```
DAG 를 만들 때마다 복사해서 쓰는 기초 뼈대 코드
Task(내용물) 와 순서(Dependency) 만 갈아끼우는 방식
```

---

---

# ① 저장 위치 — 가장 중요

```
Airflow 는 dags/ 폴더 안의 .py 파일만 인식

Docker 환경:    ./airflow/dags/my_dag.py
Standalone:     ~/airflow/dags/my_dag.py

저장 후 약 30초~1분 후 Web UI (localhost:8082) 에 자동 반영
반영 안 되면 docker-compose restart
```

---

---

# ② 표준 템플릿 — Copy & Paste

```python
import pendulum
from airflow import DAG
from airflow.operators.empty import EmptyOperator    # 구 DummyOperator
from airflow.operators.python import PythonOperator

# ① 공통 설정 — 모든 태스크가 물려받을 속성
default_args = {
    'owner'          : 'data_team',              # 책임자 (에러 나면 이 팀)
    'retries'        : 3,                        # 실패 시 재시도 횟수
    'retry_delay'    : pendulum.duration(minutes=5),     # 재시도 간격 (기본값도 5분이지만 명시 권장)
    'depends_on_past': False,                    # 어제 실패해도 오늘은 실행
}

# ② DAG 정의
with DAG(
    dag_id    = 'my_first_dag',                  # 파일명과 일치시키는 게 국룰
    description= 'DAG 표준 템플릿',
    default_args= default_args,
    schedule  = '0 9 * * *',                     # 매일 오전 9시 (KST)
    start_date= pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"),
    catchup   = False,                           # 과거 밀린 실행 금지
    tags      = ['template'],
) as dag:

    # ③ 로직 함수 정의
    def print_hello():
        print("Hello Airflow!")

    # ④ 태스크 정의
    start = EmptyOperator(task_id='start')

    task1 = PythonOperator(
        task_id        = 'python_task',
        python_callable= print_hello,
    )

    end = EmptyOperator(task_id='end')

    # ⑤ 실행 순서
    start >> task1 >> end
```

>schedule은 혹시 업데이트랑 엇갈릴까봐 여유있게 00시 05분으로 정리

---

---
# ③ default_args — 공통 설정 상속 (with DAG 밖 별도 딕셔너리)

```
태스크마다 일일이 설정 반복 방지
DAG 에 한 번 등록 → 모든 태스크가 자동 상속
```

|속성|설명|권장값|
|---|---|---|
|`owner`|책임자 명시|팀명 or 본인 ID|
|`retries`|실패 시 재시도 횟수|`3`|
|`retry_delay`|재시도 대기 시간|`pendulum.duration(minutes=5)`|
|`depends_on_past`|어제 실패 시 오늘 실행 여부|`False`|
|`email_on_failure`|실패 시 이메일 알림|`False`|
|`email_on_retry`|재시도 시 이메일 알림|`False`|

## 각 속성 상세 

```python
default_args = {
    # 책임자
    'owner': 'data_team',
    # → 에러 발생 시 Web UI 에서 누구 담당인지 표시
    # → 팀 협업 시 담당자 빠르게 파악 가능

    # 재시도
    'retries': 3,
    # → 태스크 실패 시 자동으로 3번 재시도
    # → 0 이면 재시도 없이 바로 실패 처리

    'retry_delay': pendulum.duration(minutes=5),
    # → 재시도 사이 대기 시간 (안 적으면 기본값 5분)
    # → 명시적으로 적는 게 국룰 (나중에 수정하기 쉬움)

    # 과거 의존성
    'depends_on_past': False,
    # → True: 어제 태스크가 성공해야 오늘 실행 (순서 보장 필요할 때)
    # → False: 어제 실패해도 오늘은 무조건 실행 (독립 실행) ← 보통 이걸 씀

    # 이메일 알림
    'email_on_failure': False,
    # → True: 태스크 최종 실패 시 이메일 전송
    # → False: 알림 없음 ← Slack 연동 등 다른 방법 쓸 때 끔
    # → 이메일 설정이 안 돼있으면 True 해도 에러만 남 → False 권장

    'email_on_retry': False,
    # → True: 재시도할 때마다 이메일 전송
    # → retries=3 이면 최대 3번 메일 → 메일 폭탄 → False 권장
}
```

```
이메일 알림 요약:
  email_on_failure=True  → 최종 실패(retry 다 소진)할 때만 1번
  email_on_retry=True    → 재시도할 때마다 (retries=3 이면 최대 3번)
  → 둘 다 False 가 기본 권장
     Slack 알림이나 별도 모니터링 쓸 때는 어차피 여기서 안 씀
```

---
---
# ④ DAG 정의 핵심 설정 — with DAG(...) 안

## start_date — pendulum 필수 ⭐️

```
❌ 절대 금지: datetime.now()
   기준점이 매번 바뀌어 실행이 꼬임

✅ 필수: pendulum.datetime(..., tz="Asia/Seoul")
   이유: 타임존 없으면 Airflow 가 UTC 기준으로 실행
         '0 9 * * *' 이 UTC 9시 = KST 18시 에 실행됨
         tz="Asia/Seoul" 명시 → KST 9시에 정확히 실행
```


```python
# ❌
from datetime import datetime
start_date = datetime(2026, 1, 1)           # 타임존 없음 → UTC 기준

# ✅
import pendulum
start_date = pendulum.datetime(2026, 1, 1, tz="Asia/Seoul")  # KST 명시
```

## catchup — 항상 False

```
True  (기본값):  start_date 부터 오늘까지 안 돈 DAG 전부 소급 실행
                 → 처음 켰을 때 수백 개 한꺼번에 실행 → 서버 폭발 💥

False (권장):    과거는 무시하고 오늘 스케줄부터 실행
                 → 항상 catchup=False 명시하는 습관
```

## schedule — cron 표현식

```python
schedule = '0 9 * * *'    # 매일 09:00
schedule = '0 2 * * *'    # 매일 02:00
schedule = '@daily'        # 매일 자정 (= '0 0 * * *')
schedule = '@hourly'       # 매시 정각
schedule = None            # 수동 실행만
```

>[[Linux_Scheduling_Crontab]] & [[Airflow_Scheduling]] 참고 

## tags — Web UI 필터링용

```
DAG 가 많아지면 목록에서 찾기 힘듦
tags 로 분류해두면 Web UI 상단 필터로 빠르게 검색 가능

어떤 문자열이든 자유롭게 넣으면 됨
리스트라서 여러 개 가능
```

```python
tags = ['train']                    # 프로젝트 이름
tags = ['train', 'schedule']        # 프로젝트 + 기능
tags = ['etl', 'daily']             # 용도 + 주기
tags = ['ml', 'preprocessing']      # 도메인 + 단계
tags = ['data_team', 'prod']        # 팀 + 환경

# 이 프로젝트 패턴
tags = ['train', 'delay']           # train_delay_dag
tags = ['train', 'schedule']        # train_schedule_dag
```

```
실무 팁:
  팀명, 환경(dev/prod), 도메인, 주기 조합으로 태그 규칙 정하는 게 좋음
  예) ['data_team', 'prod', 'daily']
  → 나중에 "prod 환경 DAG 만 보기" 필터링 가능
```





---

---

# ⑤ default_args 상속 우선순위

```
"더 구체적인 것이 이긴다"

Level 1: default_args (DAG 기본값) ← 가장 약함
Level 2: 태스크 개별 설정          ← 덮어씀
```

```python
with DAG(..., default_args={'retries': 3}) as dag:

    # 별도 설정 없음 → default_args 상속 → 재시도 3회
    task_a = EmptyOperator(task_id='A')

    # 직접 retries=5 지정 → default_args 무시 → 재시도 5회
    task_b = EmptyOperator(task_id='B', retries=5)
```

```
실무 패턴:
  90% 의 태스크가 쓸 설정 → default_args 에 몰아넣기
  특별한 태스크만 개별 설정으로 덮어쓰기
  → 코드 중복 제거 + 팀 표준화
```

---

---

# ⑥ dag_id 와 파일명 규칙

```
파일명: my_pipeline.py
dag_id: 'my_pipeline'     ← 동일하게 맞추는 게 국룰

이유:
  dag_id 만 보고 어떤 파일인지 바로 찾을 수 있음
  팀에서 관리할 때 혼선 방지
```

---

---

# 초보자가 자주 착각하는 것들

## ① print 로그가 터미널에 안 보여요

```
태스크 안의 print 는 터미널에 출력 안 됨
Web UI → 해당 태스크 클릭 → [Logs] 탭에서 확인
```

## ② 파일 저장했는데 Web UI 에 안 보여요

```
스케줄러가 인식하는 데 30초~1분 소요
기다려도 안 보이면:

docker-compose restart
또는
docker compose restart airflow
```

## ③ start_date 를 오늘로 설정했는데 바로 안 돌아요

```
Airflow 는 start_date + schedule 이 지난 후에 실행
예) start_date = 오늘, schedule = @daily
    → 내일이 되어야 첫 실행

의도가 즉시 실행이라면:
  Web UI → DAG → ▶ Trigger DAG 로 수동 실행
```

## ④ 파일명 바꿨는데 예전 DAG 가 남아있어요

```
Web UI → 예전 DAG → 우측 메뉴 → Delete DAG 로 수동 삭제
또는 docker-compose restart
```