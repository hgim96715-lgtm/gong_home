---
aliases:
  - execution_date
  - logical_date
  - 실행기준일
tags:
  - Airflow
related:
  - "[[Airflow_Scheduling]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[00_Airflow_HomePage]]"
---

# Airflow_Execution_Date — 실행 기준일 개념

## 한 줄 요약

```
Airflow 는 "오늘 실행"하지만 "어제 날짜"를 기준으로 동작한다
처음 보면 버그처럼 보이지만 의도된 설계
```

---

---

# 왜 헷갈리나?

```
직관적 생각:
  "@daily" → 오늘(3월 15일) 자정에 실행 → execution_date = 3월 15일

Airflow 실제 동작:
  "@daily" → 오늘(3월 15일) 자정에 실행 → execution_date = 3월 14일 (어제!)

→ 처음 보면 버그처럼 느껴지지만 의도된 설계
```

---

---

# ① Airflow 의 핵심 철학 — 데이터 구간 처리

```
Airflow 는 "지금 몇 시야?" 가 아니라
"어느 구간의 데이터를 처리하는 거야?" 를 기준으로 생각

@daily 스케줄:
  3월 15일 00:00 에 실행 → "3월 14일 하루치 데이터" 처리
  3월 16일 00:00 에 실행 → "3월 15일 하루치 데이터" 처리

  → 실행 시점(3월 15일) 이 아니라
    처리 대상 구간의 시작(3월 14일 = 어제) 이 기준

왜 이렇게 설계했나:
  "어제 하루치 데이터가 다 쌓인 걸 오늘 자정에 처리"
  → 오늘 자정이 되어야 어제 데이터가 완성됨
  → ETL 배치 처리의 자연스러운 흐름을 표현한 것
```

---

---

# ② 용어 정리 — 버전별 변경

```
Airflow 2.2 이전:  execution_date
Airflow 2.2 이후:  logical_date   (이름만 바뀜, 의미 동일)

둘 다 같은 것 = "이번 실행이 담당하는 데이터 구간의 시작 시각"
```

|용어|버전|의미|
|---|---|---|
|`execution_date`|2.2 이전|이번 실행의 논리적 기준 시각|
|`logical_date`|2.2 이후|execution_date 와 동일 (이름만 변경)|
|`data_interval_start`|2.2 이후|처리 대상 데이터 구간 시작|
|`data_interval_end`|2.2 이후|처리 대상 데이터 구간 끝|

```
data_interval_start = logical_date = (구) execution_date
data_interval_end   = data_interval_start + schedule 간격
```

---

---

# ③ @daily 기준 실제 예시

```
schedule = '@daily'  (= 매일 자정 실행)

실행 시각          data_interval_start   data_interval_end
3월 15일 00:00  →  3월 14일 00:00    ~   3월 15일 00:00
3월 16일 00:00  →  3월 15일 00:00    ~   3월 16일 00:00
3월 17일 00:00  →  3월 16일 00:00    ~   3월 17일 00:00

→ "오늘 자정에 실행" = "어제 하루치 데이터 처리"
```

```
왜 이렇게 설계했나:
  "어제 하루치 데이터가 다 쌓인 걸 오늘 자정에 처리"
  → 오늘 자정이 되어야 어제 데이터가 완성됨
  → ETL 배치 처리의 자연스러운 흐름을 표현한 것
```

---

---
# ④ **context — Airflow 자동 주입

```
Airflow 가 태스크 실행 시 자동으로 만들어서 주입하는 딕셔너리

내부 동작:
  태스크 실행 →
  {'ds': '2026-03-14', 'logical_date': ..., 'run_id': ...} 생성 →
  함수 인자로 자동 주입

**context 의 의미:
  ** = 딕셔너리를 키워드 인자로 풀어서 받는 파이썬 문법
  → context['ds'] 로 꺼내 쓸 수 있음

@task 데코레이터 → provide_context 없어도 자동 주입
PythonOperator  → provide_context=True 명시 필수
```

---
---

# ⑤ 코드에서 날짜 사용하는 법

## 과거 방식 vs 현재 방식 ⭐️

```
context['logical_date'] 방식이 인터넷에 많이 나오는데
Airflow 2.2+ 에서는 더 짧고 깔끔한 방법이 있음
```


```python
# ❌ 과거 방식 (Airflow 1.x ~ 2.0 초반)
@task
def my_task(**context):
    logical_date = context['logical_date']              # 딕셔너리에서 직접 꺼냄
    kst_date = pendulum.instance(logical_date).in_timezone('Asia/Seoul')
    # → 일반 datetime 이라 pendulum.instance() 로 감싸야 했음

# ✅ 현재 방식 (Airflow 2.2+)
@task
def my_task(logical_date=None, **context):              # 파라미터에 이름만 적으면 자동 주입
    kst_date = logical_date.in_timezone('Asia/Seoul')   # 이미 pendulum 객체 → 바로 사용
```

```
두 가지 이유로 더 짧아짐:

① @task 파라미터 자동 주입
  함수 파라미터에 logical_date=None 이라고 이름만 적으면
  Airflow 가 context 딕셔너리에서 자동으로 꺼내서 주입
  → context['logical_date'] 수동으로 꺼낼 필요 없음

② Airflow 2.2+ 는 이미 pendulum 객체로 전달
  logical_date 가 이미 pendulum 객체로 들어옴
  → pendulum.instance() 로 한 번 더 감쌀 필요 없음
  → .in_timezone() 바로 사용 가능
```

## TaskFlow API — 현재 방식 @task

```python
import pendulum
from airflow.decorators import dag, task

@dag(schedule='@daily', start_date=pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"), catchup=False)
def my_dag():

    @task
    def process(logical_date=None, **context):          # 파라미터에 이름만 적기
        # logical_date 는 이미 pendulum 객체 → 바로 타임존 변환
        kst_date = logical_date.in_timezone('Asia/Seoul')
        run_ymd  = kst_date.strftime('%Y%m%d')          # '20260314'

dag = my_dag()
```

## Classic 방식 (PythonOperator)


```python
def process(**kwargs):
    # kwargs 에서 꺼내기
    logical_date = kwargs['logical_date']
    ds           = kwargs['ds']             # 'YYYY-MM-DD'
    ds_nodash    = kwargs['ds_nodash']      # 'YYYYMMDD'

task = PythonOperator(
    task_id        = 'process',
    python_callable= process,
    provide_context= True,     # kwargs 에 context 주입
)
```

## Jinja 템플릿 (BashOperator 등)

```python
task = BashOperator(
    task_id     = 'run_script',
    bash_command= 'python job.py {{ ds }} {{ ds_nodash }}',
    #                              ↑ 2026-03-14   ↑ 20260314
)
```

---

---

# ⑤ 자주 쓰는 context 변수

| 변수                    | 타입       | 예시 값                        | 설명                  |
| --------------------- | -------- | --------------------------- | ------------------- |
| `ds`                  | str      | `'2026-03-14'`              | 처리 기준일 (YYYY-MM-DD) |
| `ds_nodash`           | str      | `'20260314'`                | 처리 기준일 (YYYYMMDD)   |
| `logical_date`        | datetime | `2026-03-14 00:00:00+09:00` | 처리 기준 시각            |
| `data_interval_start` | datetime | `2026-03-14 00:00:00`       | 데이터 구간 시작           |
| `data_interval_end`   | datetime | `2026-03-15 00:00:00`       | 데이터 구간 끝            |
| `run_id`              | str      | `'scheduled__2026-03-14'`   | 실행 ID               |
| `next_ds`             | str      | `'2026-03-15'`              | 다음 실행 기준일           |
| `prev_ds`             | str      | `'2026-03-13'`              | 이전 실행 기준일           |

---

---
# ⑦ pendulum.instance() — KST 날짜 변환 패턴


```
Airflow 2.2+ 에서는 logical_date 가 이미 pendulum 객체로 전달됨
→ pendulum.instance() 감싸기 불필요
→ .in_timezone() 바로 사용 가능

pendulum.instance() 는 옛날 코드(Airflow 2.0 이전) 에서 쓰던 방식
인터넷 예제에 많이 나오지만 현재는 불필요
```


```python
# ❌ 과거 방식 — pendulum.instance() 로 감싸야 했음
kst_date = pendulum.instance(logical_date).in_timezone('Asia/Seoul')

# ✅ 현재 방식 (Airflow 2.2+) — 이미 pendulum 객체라 바로 사용
kst_date = logical_date.in_timezone('Asia/Seoul')
run_ymd  = kst_date.strftime('%Y%m%d')   # '20260314'
```

## 패턴 A — 전날 데이터 분석 (train_delay_dag)

```
@daily 스케줄 → logical_date = 어제
→ 어제 데이터를 처리하는 경우 logical_date 그대로 사용
```

```python
@task
def run_delay_analysis(logical_date=None, **context):
    # logical_date = 어제 (Airflow @daily 의 기본 동작)
    # subtract(days=1) 하면 그저께가 되니 주의!
    yesterday = logical_date.in_timezone('Asia/Seoul').strftime('%Y%m%d')  # '20260314'

    print(f"전날 지연 분석 대상: {yesterday}")
    plan_items = train_info.get_train_schedule(yesterday)
```

```
3월 15일 02:00 에 실행
  logical_date = 3월 14일 → yesterday = '20260314'
  → 3월 14일 하루치 지연 데이터 분석
```

## 패턴 B — 오늘 데이터 수집 (train_schedule_dag)

```
@daily 스케줄 → logical_date = 어제
→ 오늘 데이터가 필요하면 하루를 더하거나 pendulum.now() 사용
```


```python
@task
def collect_schedule(logical_date=None, **context):
    # 방법 1: logical_date + 1일 = 오늘
    kst_date = logical_date.in_timezone('Asia/Seoul')
    today    = kst_date.add(days=1).strftime('%Y%m%d')    # '20260315' ← 오늘

    # 방법 2: 실제 오늘 날짜 직접 사용 (더 직관적)
    today = pendulum.now('Asia/Seoul').strftime('%Y%m%d')  # '20260315'

    print(f"오늘 운행계획 수집: {today}")
    items = train_info.get_train_schedule(today)
```

## 두 패턴 비교

```
                     logical_date 기준           실제 오늘 기준
train_delay_dag      logical_date (어제) ✅       pendulum.now() - 1일
train_schedule_dag   logical_date + 1일           pendulum.now() ✅

어제 데이터 처리 → logical_date 그대로 (Airflow 의도와 일치)
오늘 데이터 처리 → pendulum.now() 가 더 직관적
```

```
왜 KST 변환이 필요한가:
  logical_date 가 UTC 00:00 으로 오는 경우
  UTC 00:00 = KST 09:00 → 날짜가 하루 차이날 수 있음
  → 날짜 기반 DB 조회는 반드시 .in_timezone('Asia/Seoul') 후 사용
```

---
---
# ⑧ ds vs 오늘 날짜

```
ds    = 처리 기준일 (어제)
오늘  = 실제 실행 시각

둘을 혼동하면 쿼리 조건이 잘못됨
```


```python
@task
def process(**context):
    ds    = context['ds']                                    # 처리 기준일 (어제)
    today = pendulum.now('Asia/Seoul').to_date_string()      # 실제 오늘

    print(f"처리 기준일(어제): {ds}")     # 2026-03-14
    print(f"실제 오늘:         {today}")  # 2026-03-15
```

```
언제 뭘 쓰나:
  ds      → DB 에서 "어제 데이터" 조회  WHERE DATE(created_at) = '{{ ds }}'
  오늘    → 로그에 실행 시각 기록 / 파일명에 실제 날짜 붙일 때
```


---
---
# 초보자가 자주 하는 실수

##  ① ds 로 조회했는데 데이터가 0건이에요

```
@daily 에서 ds = 어제 날짜
오늘 데이터를 조회하려고 WHERE date = '{{ ds }}' 하면 당연히 0건

→ 배치 ETL 은 보통 어제 데이터를 오늘 처리 → ds 가 맞음
   오늘 데이터가 필요하면 pendulum.now('Asia/Seoul') 사용
```

## ② 수동 실행(Trigger) 하면 날짜가 이상해요

```
수동 Trigger 시 logical_date = Trigger 누른 시각
→ 원하는 날짜로 실행하려면 Trigger DAG with config 에서 날짜 지정
```

## ③ catchup=True 로 과거 실행 시 날짜가 계속 바뀌어요

```
catchup 으로 과거 실행하면 각각의 ds 가 해당 날짜로 세팅됨
→ 의도된 동작 (각 날짜의 데이터를 순서대로 처리)
```

