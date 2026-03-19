---
aliases:
  - CPU
  - 코어
  - 스레드
  - LoadAverage
tags:
  - CS
related:
  - "[[00_CS_HomePage]]"
  - "[[CS_RAM_Memory]]"
  - "[[Linux_Memory]]"
---
# CS_CPU_Architecture — CPU 코어와 스레드

## 한 줄 요약

```
코어 = 물리적 연산 장치 (입)
스레드 = 논리적 작업 단위 (손)
4코어 8스레드 = 입 4개 / 손 8개
```

---

---

# ① 코어 vs 스레드

## 코어 (Physical Core) = 입

```
실제로 물리적으로 존재하는 연산 장치
4코어 = 밥(데이터) 을 4숟가락 동시에 먹을 수 있음
```

## 스레드 (Logical Thread) = 손

```
인텔 하이퍼스레딩 기술 — 코어 1개가 2개인 척 동작
4코어 8스레드 = 입 4개, 손 8개 → 손이 쉴 새 없이 입에 넣어줌

OS 는 스레드 개수를 CPU 개수로 인식
→ 4코어 8스레드 노트북 → 작업 관리자 그래프 8개
```

## 내 맥북 확인

```bash
sysctl -n hw.physicalcpu hw.logicalcpu
# 결과: 10  10
```

```
결과가 10 10 (1:1 비율) 이면 Apple Silicon (M1/M2/M3)
  Intel:         4코어 → 8스레드 (하이퍼스레딩으로 2배)
  Apple Silicon: 10코어 → 10스레드 (진짜 코어만 사용)

Apple Silicon 은 코어를 두 종류로 나눔:
  P-Core (Performance) 고성능 작업 — 동영상 편집 / 컴파일
  E-Core (Efficiency)  저전력 작업 — 백그라운드 / 웹서핑

결론: 내 맥북은 입 10개 손 10개 — 정직한 10인분
```

---

---

# ② Load Average — 서버 부하 읽기

```
공식: Load Average / 논리 스레드 수 = 사용률

4코어 8스레드 서버 기준:
```

|Load Average|의미|상태|
|---|---|---|
|4.0|손 8개 중 4개 가동|50% — 여유 있음|
|8.0|손 8개 전부 가동|100% — 풀가동|
|16.0|8개 일하고 8개 대기|200% — 과부하 ⚠️|

```bash
uptime
# 14:30:00 up 5 days, load average: 1.20, 0.85, 0.70
#                                    ↑1분  ↑5분  ↑15분
```

```
uptime Load Average 볼 때:
  Intel 서버 → 논리 스레드 수 기준
  Apple Silicon / ARM 서버 → 코어 수 기준 (1:1 이라 동일)

vCPU (클라우드):
  AWS / GCP 에서 서버 빌릴 때 나오는 단위
  vCPU = 논리 스레드 1개
  2 vCPU 서버 → Load Average 2.0 넘으면 위험
```

---

---

# ③ 컨텍스트 스위칭 (Context Switching)

```
CPU 는 사실 한 번에 한 가지 일만 할 수 있음

그런데 음악 들으면서 코딩이 가능한 이유:
  "음악 0.001초 → 코딩 0.001초 → 카톡 0.001초"
  엄청 빠르게 번갈아 가며 처리 → 동시에 되는 것처럼 보임
```

```
컨텍스트 스위칭 비용:
  일을 바꿀 때마다 현재 상태 저장 → 다음 작업 상태 불러오기
  이 과정에서 오버헤드(시간 낭비) 발생

교훈:
  스레드를 무작정 많이 만든다고 빨라지지 않음
  너무 많으면 일은 안 하고 자리 바꾸느라 시간 다 씀
  적절한 스레드 수 = CPU 코어 수에 맞게 조정
```

---

---

# DE 관점 — Spark 와 연결

```
Spark executor 설정:
  spark.executor.cores = 4   ← Executor 당 코어 수
  spark.executor.instances = 3  ← Executor 개수

  총 병렬 처리 = 4 × 3 = 12 Task 동시 실행

vCPU 비용 절감:
  AWS EC2 2 vCPU 빌렸을 때
  Load Average 2.0 넘으면 처리 지연 시작
  → Spark 파티션 수 / Executor 수 조정 필요
```

---

---

# 한눈에 정리

```
코어  물리적 연산 장치 (입)
스레드 논리적 작업 단위 (손) — 하이퍼스레딩으로 코어 2배

4코어 8스레드:
  OS 는 CPU 8개로 인식
  Load Average 8.0 = 100% 풀가동
  8.0 초과 = 대기열 발생, 시스템 느려짐

vCPU = 논리 스레드 1개
  2 vCPU 서버 → Load 2.0 이하로 관리
```