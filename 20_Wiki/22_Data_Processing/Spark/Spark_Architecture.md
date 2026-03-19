---
aliases:
  - Driver
  - Executor
  - ClusterManager
  - 스파크구조
  - MasterNode
  - WorkerNode
tags:
  - Spark
related:
  - "[[Spark_Installation_Local_Docker]]"
  - "[[Spark_Concept_Evolution]]"
  - "[[Process_vs_Thread]]"
  - "[[CS_RAM_Memory]]"
---


# Spark_Architecture — Spark 구조

## 한 줄 요약

```
Driver (시키는 놈) → Cluster Manager (중개하는 놈) → Executor (일하는 놈)
```

---

---

# ① 3가지 핵심 구성요소

## Driver — 총사령관

```
위치: Master Node
역할: 우리가 짠 코드 실행
      SparkContext 생성 → 클러스터 연결
      작업을 Task 로 쪼개서 Executor 에게 배분

비유: 현장 소장 (설계도 들고 지시)
```

## Cluster Manager — 자원 관리자

```
위치: Master Node 또는 별도 서버
역할: 전체 서버의 CPU / 메모리 현황 파악
      Driver 가 "Executor 3개 필요해" 요청하면
      놀고 있는 Worker Node 에서 자원 할당

종류:
  Standalone   Spark 자체 내장 ← 우리 프로젝트 (테스트/개발용)
  YARN         Hadoop 생태계 표준
  Kubernetes   컨테이너 기반 (요즘 뜨는 추세)

비유: 인력 사무소 (채용 담당)
```

## Executor — 일꾼

```
위치: Worker Node 안에서 실행되는 프로세스
역할: Driver 가 시킨 Task 를 실제로 계산
      처리 중인 데이터를 Cache (메모리) 에 저장

비유: 현장 인부 (시킨 것만 함)
```

---

---

# ② Worker Node vs Executor — 자주 하는 착각

```
Worker Node  물리적 컴퓨터(서버) 1대  → 하드웨어
Executor     그 안에서 실행되는 프로세스 → 소프트웨어

Worker Node 1대 (32코어) 안에
  Executor (4코어짜리) 8개가 들어갈 수 있음

물리 서버(Worker) 1대 = 여러 개의 일꾼(Executor)
```

---

---

# ③ 작동 흐름

```
사용자
  ↓ spark-submit 실행
Driver 시작
  ↓ "Executor 5개 필요해"
Cluster Manager
  ↓ Worker Node 에 지시
Executor 생성 (Worker Node 안)
  ↓ Task + 코드 전송
Executor 가 Task 실행 → 결과를 Driver 에 반환
```

```python
# 코드 관점에서 보면

spark = SparkSession.builder.appName("test").getOrCreate()
# → SparkContext 생성 → Cluster Manager 에 자원 요청 → Executor 기동

df = spark.read.csv("data.csv")
df.filter(df.hvec > 0).show()
# → Driver 가 Task 로 쪼갬 → Executor 에 배분 → 결과 반환
```

---

---

# ④ PySpark 동작 원리 — Py4J

```
문제:
  우리는 Python 으로 코드 작성
  Spark 는 원래 Scala / JVM 으로 만들어짐
  → 언어가 달라서 직접 통신 불가

해결: Py4J (통역사)
  Python 코드 → Py4J → JVM (실제 Spark) 호출

흐름:
  df.filter()  ← Python 코드
      ↓ Py4J
  JVM 에서 실제 Spark filter() 실행
      ↓
  Worker Node 에서 Python Worker 프로세스 생성
  → Python UDF 등 파이썬 로직 처리
```

```
성능 이슈:
  Python → JVM 통역 과정에서 오버헤드 발생
  Scala 보다 PySpark 가 약간 느린 이유
  BUT 대부분의 작업에서 체감 차이 없음
```

---

---

# ⑤ Driver OOM — 자주 하는 실수

```python
# ❌ collect() 남용
result = df.collect()   # 모든 데이터가 Driver 메모리로 쏠림
# 수억 건 데이터면 Driver OOM 발생

# ✅ 필요한 것만
result = df.limit(100).collect()
# 또는 파일로 저장
df.write.parquet("output/")
```

```
collect() 하면:
  Executor (분산) → Driver (한 곳) 로 전부 모임
  Driver 메모리가 작으면 OOM

교훈:
  분산 처리 결과를 한 곳으로 모으는 건 비싼 작업
  꼭 필요할 때만 / 작은 데이터만 collect()
```

---

---

# 한눈에 정리

|구성요소|위치|역할|비유|
|---|---|---|---|
|Driver|Master Node|코드 실행 / Task 분배|현장 소장|
|Cluster Manager|Master Node|자원 할당|인력 사무소|
|Executor|Worker Node|Task 실행 / 캐싱|현장 인부|

```
면접 한 줄:
  "Driver 는 뇌를 쓰고 Executor 는 힘을 씁니다.
   Driver 가 작업을 Task 로 쪼개서
   Cluster Manager 를 통해 Worker Node 의 Executor 에 배분합니다."
```