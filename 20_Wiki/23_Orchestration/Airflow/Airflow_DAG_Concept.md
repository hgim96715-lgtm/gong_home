---
aliases:
  - DAG 개념
  - 방향성 비순환 그래프
  - Workflow
  - Task vs Instance
tags:
  - Airflow
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Why_Airflow]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Airflow_Operators]]"
---
# DAG_Concept — DAG 란 무엇인가

## 한 줄 요약

```
Directed Acyclic Graph
한쪽 방향으로만 흐르고 (Directed)
절대 순환하지 않는 (Acyclic)
작업의 흐름 (Graph)
```

---

---

# ① 왜 순환이 없어야 하는가

```
A → B → A → B → A ...  (무한 루프)
→ 작업이 영원히 끝나지 않음

DAG 는 이 구조를 수학적으로 금지
→ 모든 작업은 반드시 끝이 있음을 보장
→ 데이터가 준비되기 전에 분석이 시작되는 실수 방지
```

---

---

# ② 구성 요소 — Node 와 Edge

```
Node (Task):   그래프의 점 — 실행할 작업 하나
               Python / Bash / SQL / API 호출 등

Edge (Dependency): 그래프의 선 — 작업 실행 순서
                   >> 화살표로 표현
```

```python
# Task 3개 / Edge 2개
extract >> transform >> load

# 시각적으로:
extract ──→ transform ──→ load
```

```
실무 예시:
  매일 9시에 (Schedule)
  주문 데이터 수집 (Task A: extract)
  → 정제 (Task B: transform)
  → DB 저장 (Task C: load)

이 하나의 흐름 전체 = DAG
```

---

---

# ③ Task vs Task Instance ⭐️

```
가장 헷갈리는 개념
Airflow 로그 분석의 핵심
```

|구분|Task|Task Instance|
|---|---|---|
|비유|붕어빵 틀|오늘 구운 팥붕어빵|
|정의|코드에 적힌 작업 정의|특정 날짜에 실행된 Task|
|상태|없음 (그냥 코드)|Success / Failed / Running / Retry|
|예시|`extract_task = PythonOperator(...)`|2026-03-19 의 extract_task 실행 결과|

```python
# Task 정의 (붕어빵 틀)
extract_task = PythonOperator(
    task_id="extract",
    python_callable=fetch_data,
)

# Task Instance (오늘 구운 붕어빵)
# → 2026-03-19 에 실행된 extract_task
# → 상태: Success / Failed / Running
```

```
Task Instance 가 실패하면:
  "2026-03-19 자 extract_task" 가 실패한 것
  코드(Task) 는 그대로
  다음 날(2026-03-20) 자 Instance 는 성공할 수 있음
  (어제 데이터 문제였을 수 있으니까)
```

---

---

# ④ DAG 기본 구조

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG(
    dag_id="my_first_dag",         # DAG 고유 이름
    schedule_interval="0 9 * * *", # 매일 9시
    start_date=datetime(2026, 1, 1),
    catchup=False,                 # 과거 누락 실행 안 함
) as dag:

    task_a = PythonOperator(task_id="extract",   python_callable=extract)
    task_b = PythonOperator(task_id="transform", python_callable=transform)
    task_c = PythonOperator(task_id="load",      python_callable=load)

    task_a >> task_b >> task_c   # 순서 지정
```

---

---

# ⑤ 의존성 패턴

```python
# 순차 실행
a >> b >> c

# 병렬 실행 후 합치기 (fan-out → fan-in)
a >> [b, c] >> d
# a 끝나면 b, c 동시에 / b, c 모두 끝나면 d 시작

# 역방향 (같은 의미)
c << b << a
```

---

---

# ⑥ 자주 하는 착각

```
"DAG 안에 for 루프 넣으면 안 되나요?"
  → DAG 자체는 루프 불가 (Acyclic 원칙)
  → Task 내부 Python 코드는 for 문 사용 가능
  → 여러 Task 를 동적으로 만들고 싶으면 Dynamic DAG 사용

"Task Instance 실패 = Task 실패?"
  → 아님
  → "2026-03-19 자 Task Instance" 가 실패한 것
  → 코드(Task) 는 정상 / 데이터 문제일 수 있음
  → 해당 날짜 Instance 만 재실행하면 됨

"파이프라인 짰어? = DAG 그렸어?"
  → 실무에서 같은 말로 사용됨
```

---

---

# 한눈에 정리

```
DAG = 작업(Task) 들의 흐름
  방향 있음 (Directed)    →  순서가 있음
  순환 없음 (Acyclic)     →  무한 루프 없음

Task      코드에 적힌 작업 정의 (붕어빵 틀)
Instance  특정 날짜에 실행된 결과 (오늘 구운 붕어빵)

>> 연산자로 순서 지정
a >> b >> c           순차
a >> [b, c] >> d      병렬 후 합치기
```