---
aliases:
  - execution_date
  - logical_date
  - 실행기준일
  - pendulum
  - ds
tags:
  - Airflow
related:
  - "[[Airflow_Scheduling]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[00_Airflow_HomePage]]"
---
# Airflow_Execution_Date — 실행 기준일

## 한 줄 요약

```
Airflow 는 "오늘 실행"하지만 "어제 날짜"를 기준으로 동작한다
"지금 몇 시야?" 가 아니라 "어느 구간의 데이터를 처리하는 거야?" 기준
```

---

---

# ① 왜 어제 날짜인가

```
직관: "@daily" → 3월 15일 자정 실행 → ds = 3월 15일
실제: "@daily" → 3월 15일 자정 실행 → ds = 3월 14일 (어제!)
```

```
이유:
  3월 15일 자정 = "3월 14일 하루치 데이터가 막 완성된 시각"
  → 3월 14일 데이터를 지금 처리하는 것이 자연스러운 ETL 흐름

  @daily 스케줄:
    3월 15일 00:00 실행 → 3월 14일 데이터 처리
    3월 16일 00:00 실행 → 3월 15일 데이터 처리
    3월 17일 00:00 실행 → 3월 16일 데이터 처리
```

---

---

# ② 용어 정리

```
Airflow 2.2 이전: execution_date
Airflow 2.2 이후: logical_date  (이름만 변경, 의미 동일)

둘 다 = "이번 실행이 처리하는 데이터 구간의 시작 시각"
```

|용어|의미|
|---|---|
|`execution_date`|(구) 실행 기준 시각|
|`logical_date`|(신) execution_date 와 동일|
|`data_interval_start`|처리 구간 시작 = logical_date|
|`data_interval_end`|처리 구간 끝 = start + schedule 간격|
|`ds`|data_interval_start 의 날짜 문자열 (YYYY-MM-DD)|

```
@daily 기준 예시:

  실행 시각             data_interval_start   data_interval_end   ds
  3/15 00:00  →        3/14 00:00        ~   3/15 00:00         '2026-03-14'
  3/16 00:00  →        3/15 00:00        ~   3/16 00:00         '2026-03-15'

  @hourly 기준 예시:
  3/15 09:00  →        3/15 08:00        ~   3/15 09:00         '2026-03-15'
```

---

---

# ③ context — Airflow 자동 주입 딕셔너리

```
Airflow 가 태스크 실행 시 자동으로 만들어서 함수에 주입하는 딕셔너리
실행 기준일 / 구간 / run_id 등을 담고 있음
```

## 자주 쓰는 context 변수

|변수|타입|예시 값|설명|
|---|---|---|---|
|`ds`|str|`'2026-03-14'`|처리 기준일 YYYY-MM-DD|
|`ds_nodash`|str|`'20260314'`|처리 기준일 YYYYMMDD|
|`logical_date`|pendulum|`2026-03-14 00:00:00+09:00`|처리 기준 시각 (pendulum 객체)|
|`data_interval_start`|pendulum|`2026-03-14 00:00:00`|구간 시작 (pendulum 객체)|
|`data_interval_end`|pendulum|`2026-03-15 00:00:00`|구간 끝 (pendulum 객체)|
|`run_id`|str|`'scheduled__2026-03-14'`|실행 ID|
|`next_ds`|str|`'2026-03-15'`|다음 실행 기준일|
|`prev_ds`|str|`'2026-03-13'`|이전 실행 기준일|

## context 꺼내는 방법

```python
# ✅ TaskFlow API — 파라미터 이름으로 자동 주입 (권장)
@task
def my_task(logical_date=None, data_interval_start=None, data_interval_end=None):
    kst = logical_date.in_timezone("Asia/Seoul")   # 바로 pendulum 메서드 사용
    start = data_interval_start.in_timezone("Asia/Seoul").start_of("hour")
```

```text
TaskFlow API 자동 주입 원리:
  Airflow 가 함수 파라미터 이름을 스캔
  logical_date / data_interval_start / ds / run_id 같은 예약어 발견
  → context 딕셔너리에서 자동으로 꺼내서 직접 주입

  context['logical_date'] 로 수동으로 꺼낼 필요 없음
  logical_date / data_interval_start 는 이미 pendulum.DateTime 객체
  → .in_timezone() / .start_of() 바로 사용 가능

  ds 처럼 문자열이 필요한 건 여전히 context['ds'] 로 꺼내거나
  파라미터에 ds=None 으로 선언하면 문자열로 자동 주입됨
```

```python
# ds 도 파라미터로 직접 받기
@task
def my_task(logical_date=None, ds=None, ds_nodash=None):
    print(ds)        # '2026-03-14'   ← 문자열
    print(ds_nodash) # '20260314'     ← 문자열
    kst = logical_date.in_timezone("Asia/Seoul")  # pendulum 객체
```

```
⚠️ PythonOperator (Classic 방식) 는 다름
  @task 자동 주입은 TaskFlow API 전용
  PythonOperator 에서는 **kwargs 로 받고 kwargs['logical_date'] 꺼내기
```

```python
# PythonOperator — provide_context=True + **kwargs 필수
def my_func(**kwargs):
    logical_date = kwargs['logical_date']
    ds           = kwargs['ds']
    kst = logical_date.in_timezone("Asia/Seoul")

task = PythonOperator(
    task_id        = 'my_task',
    python_callable= my_func,
    provide_context= True,   # ← 없으면 kwargs 에 context 주입 안 됨
)
```

```text
비교:

  @task (TaskFlow):
    파라미터 이름만 적으면 자동 주입
    def my_task(logical_date=None):  ← 끝

  PythonOperator (Classic):
    provide_context=True 명시 필요
    함수에서 **kwargs['logical_date'] 로 꺼내야 함
```

---

---

# ④ pendulum — Airflow 가 쓰는 시간 라이브러리

```
Airflow 2.x 는 datetime 대신 pendulum 을 사용
logical_date / data_interval_start 는 모두 pendulum 객체로 전달
→ pendulum 메서드를 바로 사용 가능
```

## .in_timezone() — 시간대 변환

```python
@task
def my_task(data_interval_start=None, **context):
    # UTC → KST 변환
    kst = data_interval_start.in_timezone("Asia/Seoul")
    print(kst)   # 2026-03-23 09:00:00+09:00

    # 문자열로 변환
    kst.strftime("%Y-%m-%d %H:%M:%S")  # '2026-03-23 09:00:00'
    kst.strftime("%Y%m%d")             # '20260323'
    kst.to_date_string()               # '2026-03-23'
```

```
in_timezone() 이 필요한 이유:
  data_interval_start 는 UTC 기준으로 들어옴
  DB 에 KST 로 저장된 데이터 조회 시 반드시 KST 로 변환
  UTC 그대로 쓰면 9시간 앞 구간 조회 → 데이터 엉킴
```

## .start_of() — 정각으로 맞추기 ⭐️

```python
@task
def aggregate(data_interval_start=None, data_interval_end=None, **context):
    start_time = (
        data_interval_start
        .in_timezone("Asia/Seoul")
        .start_of("hour")              # 분/초를 00:00 으로 맞춤
        .strftime("%Y-%m-%d %H:%M:%S")
    )
    end_time = (
        data_interval_end
        .in_timezone("Asia/Seoul")
        .start_of("hour")
        .strftime("%Y-%m-%d %H:%M:%S")
    )
    # 2026-03-23 09:00:00 ~ 2026-03-23 10:00:00
```

```
start_of('hour') 가 필요한 이유:

  스케줄 자동 실행:
    data_interval_start = 2026-03-23 09:00:00  (정각 / 정상)

  수동 Trigger 실행:
    data_interval_start = 2026-03-23 09:23:47  (누른 시각 / 비정각!)

  start_of('hour') 없이 수동 실행하면:
    WHERE created_at >= '2026-03-23 09:23:47' 로 조회
    → 9시 0분~23분 데이터 누락!

  start_of('hour') 붙이면:
    항상 2026-03-23 09:00:00 으로 맞춰짐
    → 자동/수동 실행 모두 정각 기준으로 집계 (멱등성 보장)
```

## pendulum 주요 메서드

```python
dt = data_interval_start.in_timezone("Asia/Seoul")

# 단위 시작/끝으로 이동
dt.start_of("hour")    # 09:00:00  (분/초 → 00)
dt.start_of("day")     # 00:00:00  (자정)
dt.start_of("month")   # 2026-03-01 00:00:00
dt.end_of("hour")      # 09:59:59
dt.end_of("day")       # 23:59:59

# 더하기 / 빼기
dt.add(hours=1)        # 1시간 후
dt.add(days=1)         # 1일 후
dt.subtract(hours=1)   # 1시간 전
dt.subtract(days=1)    # 1일 전

# 포맷
dt.strftime("%Y-%m-%d %H:%M:%S")  # '2026-03-23 09:00:00'
dt.strftime("%Y%m%d")             # '20260323'
dt.to_date_string()               # '2026-03-23'
dt.to_datetime_string()           # '2026-03-23 09:00:00'

# 현재 시각
pendulum.now("Asia/Seoul")        # 지금 KST 시각
```

|메서드|결과|설명|
|---|---|---|
|`.start_of("hour")`|09:00:00|시 단위 정각|
|`.start_of("day")`|00:00:00|자정|
|`.start_of("month")`|03-01 00:00:00|월 첫날|
|`.add(days=1)`|+1일|더하기|
|`.subtract(hours=1)`|-1시간|빼기|
|`.in_timezone("Asia/Seoul")`|KST 변환|시간대 변환|

---

---

# ⑤ 실전 패턴 3가지

## 패턴 A — @daily: 어제 데이터 처리

```python
@task
def process_yesterday(logical_date=None, **context):
    # logical_date = 어제 → 그대로 사용
    yesterday = logical_date.in_timezone("Asia/Seoul").strftime("%Y%m%d")
    print(f"어제 데이터 처리: {yesterday}")   # '20260314'

    # DB 조회
    # WHERE DATE(created_at) = '2026-03-14'
```

## 패턴 B — @daily: 오늘 데이터 수집

```python
@task
def collect_today(logical_date=None, **context):
    # logical_date = 어제이므로 +1일 = 오늘
    kst   = logical_date.in_timezone("Asia/Seoul")
    today = kst.add(days=1).strftime("%Y%m%d")    # '20260315'

    # 또는 실제 오늘 날짜 직접 (더 직관적)
    today = pendulum.now("Asia/Seoul").strftime("%Y%m%d")
```

## 패턴 C — @hourly: 시간대별 집계 (Hospital 프로젝트)

```python
@task
def aggregate_hourly(data_interval_start=None, data_interval_end=None, **context):
    start_time = (
        data_interval_start.in_timezone("Asia/Seoul")
        .start_of("hour").strftime("%Y-%m-%d %H:%M:%S")
    )
    end_time = (
        data_interval_end.in_timezone("Asia/Seoul")
        .start_of("hour").strftime("%Y-%m-%d %H:%M:%S")
    )
    hook.run(sql, parameters=(start_time, end_time))
```

---

---

# ⑥ ds vs 오늘 날짜

```python
@task
def process(**context):
    ds    = context['ds']                              # 처리 기준일 (어제) '2026-03-14'
    today = pendulum.now("Asia/Seoul").to_date_string() # 실제 오늘 '2026-03-15'
```

```
언제 뭘 쓰나:
  ds      → DB "어제 데이터" 조회   WHERE DATE(created_at) = '{{ ds }}'
  today   → 로그 기록 / 파일명에 실제 날짜 붙일 때
```

---

---

# 자주 하는 실수

```
① ds 로 오늘 데이터 조회했더니 0건
  @daily 에서 ds = 어제 날짜
  오늘 데이터 필요하면 pendulum.now('Asia/Seoul') 사용

② 수동 Trigger 하면 날짜가 이상함
  수동 Trigger 시 logical_date = 누른 시각 (비정각)
  → .start_of('hour') 로 정각으로 맞추기
  → 또는 Trigger DAG with config 에서 날짜 직접 지정

③ UTC/KST 혼동으로 데이터 9시간 어긋남
  data_interval_start 는 UTC 기준
  → 반드시 .in_timezone("Asia/Seoul") 후 사용

④ pendulum.instance() 쓰고 있음
  Airflow 2.2+ 에서는 이미 pendulum 객체
  pendulum.instance() 감싸기 불필요
  → logical_date.in_timezone() 바로 사용
```