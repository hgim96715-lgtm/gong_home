---
aliases:
  - Airflow 아키텍처
  - Airflow 구성요소
  - Scheduler
  - Worker
  - Executor
tags:
  - Airflow
related:
  - "[[00_Airflow_HomePage]]"
  - "[[Airflow_DAG_Concept]]"
  - "[[Why_Airflow]]"
  - "[[Airflow_Installation]]"
---
# Airflow_Architecture — Airflow 구조

## 한 줄 요약

```
지시하는 놈 (Scheduler)
기록하는 놈 (MetaDB)
보여주는 놈 (Webserver)
실제로 일하는 놈 (Worker)

철저한 분업 → 안정성 + 확장성
```

---

---

# ① 구성요소 5가지

## Scheduler — 두뇌

```
DAG 파일을 주기적으로 읽어서
"언제 어떤 Task 를 실행해야 하는지" 결정
실행 시간 되면 Executor 에게 일감 전달

이게 죽으면:
  시간이 됐는데 작업이 시작 안 됨
  → Scheduler 확인
```

## Metadata DB — 기억

```
Airflow 의 모든 상태를 저장
  DAG 성공 / 실패 기록
  Task Instance 상태
  유저 정보 / 변수 / Connection

보통 PostgreSQL / MySQL 사용
```

## Executor — 배달부

```
"어떻게 실행할 것인가" 결정하는 메커니즘

종류:
  LocalExecutor      한 컴퓨터 안에서 실행 (개발/테스트용)
  CeleryExecutor     여러 컴퓨터에 분산 실행 (운영용)
  KubernetesExecutor 파드 띄워서 실행

Executor 는 직접 코드를 실행하지 않음
→ Queue 에 일감 넣어주면 Worker 가 가져감
```

## Worker — 일꾼

```
실제 Python 코드를 실행하는 프로세스
Queue 에서 일감 꺼내서 작업 수행
결과(성공/실패)를 MetaDB 에 기록

이게 문제면:
  Running 상태에서 안 넘어감
  → Worker / Executor 확인
```

## Webserver — 화면

```
우리가 보는 Airflow UI (localhost:8084)
DAG 목록 / 로그 / 수동 실행(Trigger)

얘는 실제 작업 실행 안 함
DB 에 있는 것만 보여줌

이게 죽어도:
  Scheduler 는 계속 돌아감
  화면만 안 보일 뿐
```

---

---

# ② 작동 흐름

```
1. Scheduler 가 DAG 폴더 읽음
   "9시에 돌 작업 있네?"

2. 실행 시간 → Task 상태를 'Scheduled' 로 변경
   Executor 에게 전달

3. Executor → Queue 에 일감 추가
   (Redis / RabbitMQ)

4. 놀던 Worker 가 Queue 에서 일감 꺼냄
   Python 코드 실행

5. 결과(성공/실패) → MetaDB 기록
   Webserver 에서 확인 가능
```

```
Scheduler ──→ Executor ──→ Queue ──→ Worker
                                        ↓
                              MetaDB 에 결과 기록
                                        ↓
                              Webserver 에서 확인
```

---

---

# ③ 트러블슈팅 — 어디가 아픈가?

|증상|의심 컴포넌트|
|---|---|
|DAG 가 목록에 안 보임|Webserver / DAG 파일 문법 오류|
|시간 됐는데 작업 시작 안 함|Scheduler 죽음 또는 멈춤|
|Running 에서 안 넘어감|Worker / Executor 문제|
|로그가 안 보임|Webserver / MetaDB 연결|
|어제 성공했는데 오늘 실패|Task Instance 확인 (코드 말고 데이터 문제)|

---

---

# ④ 싱글 노드 vs 멀티 노드

## 싱글 노드 — 학습 / 개발용

```
모든 컴포넌트가 서버 1대에 설치

Webserver + Scheduler + Executor + MetaDB + Worker
→ 한 컴퓨터 안에 전부

특징:
  설치 간단 (pip install 후 실행)
  Executor: LocalExecutor
  단점: 서버 죽으면 전체 중단
        작업 많아지면 한계
```

## 멀티 노드 — 운영용

```
역할별로 서버 분리

Node 1 (마스터): Webserver + Scheduler + Executor
Node 2 (저장소): MetaDB (PostgreSQL)
Node 3 (큐):    Redis / RabbitMQ
Node N (워커):   Worker × N대 (필요하면 늘리기)

Executor: CeleryExecutor / KubernetesExecutor

특징:
  확장성: 작업 많아지면 Worker 노드만 추가
  안정성: Worker 하나 죽어도 다른 Worker 대신 처리
  분업: 웹 서버 죽어도 Scheduler 계속 동작
```

---

---

# ⑤ 자주 하는 착각

```
"Executor 가 작업을 실행한다?"
  → 절반만 맞음
  → Executor 는 배달부 / 실제 실행은 Worker
  → LocalExecutor 는 같은 프로세스 안에 있어서 헷갈림

"Webserver 가 죽으면 스케줄도 멈추나?"
  → 아님
  → Webserver 는 화면(모니터)
  → 화면 꺼져도 Scheduler 는 살아서 계속 동작

"MetaDB 가 없으면?"
  → Airflow 전체 동작 불가
  → 모든 상태 기록이 MetaDB 에 있음
  → 가장 중요한 컴포넌트
```

---

---

# 한눈에 정리

|컴포넌트|역할|비유|
|---|---|---|
|Scheduler|언제 실행할지 결정|지휘자|
|MetaDB|모든 상태 저장|기록부|
|Executor|어떻게 실행할지 결정|배달 관리자|
|Worker|실제 코드 실행|현장 일꾼|
|Webserver|UI / 로그 확인|모니터|

```
죽으면 안 되는 순서:
  MetaDB > Scheduler > Worker > Executor > Webserver
```