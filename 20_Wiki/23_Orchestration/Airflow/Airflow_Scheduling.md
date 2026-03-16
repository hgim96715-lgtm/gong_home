---
aliases:
  - Airflow Scheduling
  - Data-aware Scheduling
  - Datasets
  - Catchup
  - Backfill
  - Cron
tags:
  - Airflow
  - Dataset
  - schedule
  - cron
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Linux_Scheduling_Crontab]]"
  - "[[00_Airflow_HomePage]]"
---
# Airflow_Scheduling — DAG 스케줄링

## 한 줄 요약

```
DAG 를 "언제" 실행할지 결정하는 규칙

두 가지 방식:
  Time-based   정해진 시간에 실행 (cron)
  Data-aware   특정 데이터가 만들어지면 실행 (Dataset)
```

---

---

# ① schedule 파라미터

```
Airflow 2.4 이전: schedule_interval
Airflow 2.4 이후: schedule  (통합, schedule_interval 은 Deprecated)

→ 최신 버전이면 schedule 만 사용
```

```python
with DAG(
    dag_id    = 'my_dag',
    schedule  = '0 9 * * *',   # cron 식 or 프리셋 or Dataset
    catchup   = False,
    ...
) as dag:
```

---

---

# ② Time-based — cron 표현식

```python
# cron 식 직접 작성
schedule = '0 9 * * *'    # 매일 09:00
schedule = '30 2 * * *'   # 매일 02:30
schedule = '0 9 * * 1'    # 매주 월요일 09:00
schedule = '*/10 * * * *' # 10분마다
```

## cron 프리셋 — 외워두면 편한 예약어

|프리셋|의미|cron 등가|
|---|---|---|
|`None`|스케줄 없음 — 수동 트리거만|-|
|`@once`|딱 한 번만 실행|-|
|`@hourly`|매시 정각|`0 * * * *`|
|`@daily`|매일 자정 00:00|`0 0 * * *`|
|`@weekly`|매주 일요일 자정|`0 0 * * 0`|
|`@monthly`|매월 1일 자정|`0 0 1 * *`|
|`@yearly`|매년 1월 1일 자정|`0 0 1 1 *`|

```python
# 예시
schedule = '@daily'    # '0 0 * * *' 와 동일, 가독성 좋음
schedule = None        # 수동 실행만 허용
schedule = '@once'     # 1회성 작업 / 디버깅용
```

---

---

# ③ catchup — 밀린 실행 처리

```
catchup=True (기본값):
  start_date 가 1년 전이면 오늘 DAG 켜는 순간
  지난 1년 치 (365번) 한꺼번에 실행 → 서버 폭발 💥

catchup=False (권장):
  과거는 무시, 다음 예정된 스케줄부터 실행
  → 개발할 때는 무조건 False
```

```python
with DAG(
    dag_id  = 'my_dag',
    schedule= '@daily',
    catchup = False,     # ← 항상 명시
    ...
)
```

---

---

# ④ Data-aware — Dataset (데이터 기반 실행)

```
문제: "매일 9시에 실행" 인데 앞단 데이터가 9시 5분에 완료되면?
     → 데이터 없는 채로 실행 → 에러

해결: "시간 상관없이 이 데이터가 만들어지면 실행"
     → Dataset 개념 도입 (Airflow 2.4+)
```

## Producer / Consumer 구조

```python
from airflow.datasets import Dataset

# Dataset 정의 — URI 형식의 논리적 이름표
my_data = Dataset("s3://bucket/my_data.csv")
```

```python
# Producer DAG — 데이터를 만드는 쪽
with DAG(dag_id='producer_dag', schedule='@daily', ...) as dag:
    task1 = BashOperator(
        task_id     = 'create_data',
        bash_command= 'python /app/extract.py',
        outlets     = [my_data],   # ← "나 이 데이터 업데이트했어!" 선언
    )
```

```python
# Consumer DAG — 데이터를 기다리는 쪽
with DAG(
    dag_id  = 'consumer_dag',
    schedule= [my_data],   # ← my_data 가 업데이트되면 즉시 실행
    ...
) as dag:
    task2 = BashOperator(...)
```

```
흐름:
  producer task 성공 → outlets[my_data] 업데이트 신고
  → Airflow 가 감지 → consumer DAG 자동 실행

outlets = "나 이 데이터 만들었어" 배출구
schedule = [Dataset] → "이 데이터 만들어지면 나 실행해" 구독
```

## Dataset 여러 개 (AND 조건)

```python
# dataset1, dataset2 둘 다 업데이트되어야 실행
with DAG(
    dag_id  = 'consumer_dag',
    schedule= [dataset1, dataset2],
    ...
)
```

## Dataset 주의사항

```
✅ URI 는 논리적 이름표 — 실제 파일 존재 여부 확인 안 함
   s3://bucket/file.csv 라고 해서 S3 에 접속하는 게 아님
   outlets 태스크가 "성공" 하면 업데이트됐다고 칠 뿐

✅ 일반 문자열도 가능
   Dataset("my_team_table_A") → 동작함

❌ 와일드카드 불가
   Dataset("s3://bucket/*.csv") → 안 됨 (정확히 일치해야 함)

❌ 스케줄 혼용 불가
   schedule=[my_data]  와  schedule="0 9 * * *"  동시 사용 불가

❌ 외부 툴 감지 불가
   Lambda, Spark 가 S3 에 파일 올려도 Airflow 는 모름
   반드시 Airflow Task 의 outlets 를 통해서만 트리거됨
```

---

---

# ⑤ Time-based vs Data-aware 비교

|항목|Time-based|Data-aware|
|---|---|---|
|실행 조건|정해진 시간|데이터 업데이트|
|설정 방법|`schedule='0 9 * * *'`|`schedule=[Dataset(...)]`|
|주요 용도|정기 리포트, 로그 백업|ETL 파이프라인, ML 학습|
|단점|앞단 지연 시 데이터 없이 실행|외부 트리거 감지 불가|

---

---

# 초보자가 자주 착각하는 것들

## ① schedule_interval 쓰면 안 되나?

```
작동은 하지만 Airflow 2.4+ 에서 Deprecated 경고 뜸
→ schedule 로 통일하는 습관
```

## ② Dataset URI 는 실제 경로여야 한다?

```
아니! 논리적 이름표일 뿐
Dataset("my_table") 처럼 임의 문자열도 됨
Airflow 가 S3 / DB 에 실제 접속하지 않음
```

## ③ @daily 는 한국 시간 자정인가?

```
tz 설정에 따라 다름
pendulum.datetime(..., tz="Asia/Seoul") 로 start_date 설정하면
@daily = KST 자정

tz 없으면 UTC 자정 = KST 오전 9시
→ start_date 에 항상 tz="Asia/Seoul" 명시 필수
```