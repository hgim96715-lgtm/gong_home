---
aliases:
  - 데이터 파이프라인 개념
  - 오케스트레이션
  - Airflow 장점
  - ETL
  - 한계와 유의점
tags:
  - Airflow
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_DAG_Concept]]"
  - "[[Airflow_Architecture]]"
---
# Why_Airflow — 왜 Airflow 를 쓰는가

## 한 줄 요약

```
복잡한 데이터 파이프라인의 순서 / 의존성 / 재시도를
Python 코드로 관리하는 오케스트레이터
```

---

---

# ① ETL vs ELT — 먼저 개념 잡기

## ETL (Extract → Transform → Load)

```
수집 → 가공 → 적재

과거 방식:
  원본 데이터를 가져와서
  중간 단계에서 가공(변환)한 뒤
  최종 형태로 DB 에 저장

예시:
  MySQL DB → (가공) → PostgreSQL 적재
  API 응답 → (필터링 / 변환) → DB 저장
```

## ELT (Extract → Load → Transform)

```
수집 → 적재 → 가공

현재 주류 방식 (Modern Data Stack):
  원본 데이터를 그대로 먼저 저장하고
  저장된 후에 BigQuery / Spark 같은 곳에서 가공

왜 ELT 로 바뀌었는가:
  클라우드 스토리지 저렴 → 원본 다 저장해도 됨
  BigQuery / Spark 성능 좋아짐 → 적재 후 가공해도 빠름
  원본 유지 → 나중에 다시 가공 방법 바꿔도 됨

dbt (Data Build Tool) 이 ELT 의 T 담당
```

## Airflow 와의 관계

```
Airflow 는 ETL / ELT 두 방식 모두 오케스트레이션 가능
  "언제 Extract 할지" 스케줄링
  "Transform 끝나면 Load 시작" 의존성 관리
  "실패하면 재시도" 장애 복구

직접 데이터를 가공하지 않음
→ 가공은 Spark / dbt / BigQuery 에 맡기고
→ Airflow 는 "끝났어?" 확인만
```

---

---

# ② 왜 Crontab 대신 Airflow 인가

## Crontab — 단순 반복은 충분

```bash
# 매일 새벽 2시에 스크립트 실행
0 2 * * * python3 /home/user/collect.py
```

```
문제:
  "A 가 끝나면 B 실행, B 실패하면 C 실행, B 가 이미 돌고 있으면 기다려"
  → Crontab 으로 불가능
  → Shell 스크립트 수십 개가 얽히기 시작 → 스크립트 지옥(Script Hell)
  → 오늘 데이터 왜 안 들어왔는지 찾으려면 밤샘
```

## Airflow 가 해결하는 3가지

```
① 의존성 관리:
   A 끝나야 B 시작 / B, C 병렬 실행 후 D 시작
   >> 연산자로 순서 지정

② 가시성:
   DAG 그래프로 어디서 멈췄는지 한눈에 확인
   비개발자도 흐름 이해 가능

③ 재시도 / 알림:
   실패 시 자동 재시도 (retries=3)
   Slack / 이메일 알림
```

## 면접 답변 한 줄

```
"워크플로우의 가시성 확보와 복잡한 의존성 관리 때문입니다.
 크론탭은 단순 반복에 적합하지만
 여러 작업이 얽힌 파이프라인에서는
 실패 원인 파악 / 재시도 / 순서 보장이 어렵습니다."
```

---

---

# ③ Airflow 의 핵심 장점

## Workflow as Code ⭐️

```
GUI 클릭이 아니라 Python 코드로 파이프라인 작성
  → Git 버전 관리 가능
  → 코드 리뷰 가능
  → 팀원과 협업 가능
  → 환경 재현 가능
```

## DAG 시각화

```
복잡한 의존성을 그래프로 표현
Task 별 상태 (성공 / 실패 / 실행 중) 색으로 표시
실패한 Task 만 재실행 가능
```

## 풍부한 생태계

```
Airbnb 에서 만들고 오픈소스로 공개
전 세계 수많은 Provider (AWS / GCP / Slack / PostgreSQL ...)
```

---

---

# ④ Airflow 의 한계

```
실시간 처리 불가:
  배치(Batch) 도구 → 1시간마다 / 1일마다
  실시간 스트리밍 → Kafka / Flink 사용

짧은 작업 비효율:
  작업 하나 실행할 때마다 프로세스 새로 뜸
  1초짜리 작업 수천 개 → 오버헤드가 더 큼
  차라리 while True 루프 하나가 나음

스케줄러 병목:
  DAG 수천 개, Task 수만 개 → 스케줄러 느려짐
  DAG 파일 1초마다 읽음 → top-level import 금지 이유

버전 업그레이드:
  하위 호환성 깨지는 경우 있음
  "Airflow 버전 올리다가 밤샜다" 사례 있음
```

---

---

# ⑤ 자주 하는 착각

```
"Airflow 가 데이터를 가공한다?"
  → 아님. Airflow 는 지휘자
  → 가공은 Spark / dbt / BigQuery 에 맡기고
  → Airflow 는 순서와 상태만 관리

"실시간 스트리밍도 Airflow 로?"
  → 배치 도구 / 실시간 불가
  → Kafka / Flink 써야 함

"1분마다 주식 크롤링도 Airflow 로 해도 되나요?"
  → 가능은 하지만 비효율
  → 1분마다 프로세스 띄우고 닫는 건 낭비
  → while True 루프 하나가 더 적합

"Airflow 가 느려요!"
  → DAG 파일 안에 import pandas 같은 무거운 라이브러리 있는지 확인
  → 스케줄러가 파일을 1초마다 읽음
  → top-level import 금지 → [[Airflow_Best_Practices]] 참고
```

---

---

# 한눈에 정리

```
ETL:  수집 → 가공 → 적재  (과거)
ELT:  수집 → 적재 → 가공  (현재 주류 / BigQuery / dbt)

Crontab:  단순 반복 ✅  /  복잡한 의존성 ❌
Airflow:  의존성 / 재시도 / 가시성 / Python 코드 관리

Airflow 가 직접 하지 않는 것:
  데이터 가공 (→ Spark / dbt)
  실시간 처리 (→ Kafka / Flink)

Airflow 가 하는 것:
  언제 / 어떤 순서로 / 실패하면 어떻게
```