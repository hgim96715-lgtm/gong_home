---
aliases:
  - RAM
  - 주기억장치
  - 메모리
  - OOM
  - Out Of Memory
tags:
  - CS
related:
  - "[[CS_CPU_Architecture]]"
  - "[[Spark_Concept_Evolution]]"
  - "[[00_CS_HomePage]]"
  - "[[CS_Operating_System]]"
---

# CS_RAM_Memory — RAM 과 메모리

## 한 줄 요약

```
CPU 가 작업하기 위해 데이터를 잠시 펼쳐놓는 "책상"
전원이 꺼지면 사라지는 휘발성 작업 공간
```

---

---

# ① 저장장치 계층 — 책상 비유

```
CPU (두뇌)
  책상에서 책을 읽고 계산하는 사람

RAM (주기억장치) = 책상
  작업을 위해 책을 가져와 펼쳐두는 공간
  엄청 빠름 / 전원 끄면 사라짐

HDD / SSD (보조기억장치) = 도서관 서고
  데이터를 영구 보관하는 공간
  느림 / 전원 꺼도 유지
```

```
속도 비교:
  CPU       시속 300km
  RAM       시속 200km
  SSD       시속 50km
  HDD       시속 10km

CPU 가 HDD 에서 직접 데이터 가져오면
  → 속도 차이로 CPU 가 계속 대기 (Bottleneck)

중간에 RAM 을 두면
  → 자주 쓰는 데이터 미리 RAM 에 올려둠
  → CPU 끊김 없이 빠르게 작업
```

---

---

# ② RAM 핵심 특징

## 속도 (Speed)

```
HDD 보다 수천 배 빠름
SSD 보다 수백 배 빠름

Spark 가 In-Memory 컴퓨팅으로 100배 빠른 이유:
  Hadoop MapReduce → 매 단계마다 HDD 에 씀
  Spark            → 데이터를 RAM 에 올려두고 계속 작업
  → 디스크 I/O 를 거의 안 함 → 속도 폭발
```

## 휘발성 (Volatile)

```
전기가 끊기면 데이터 사라짐

문서 작성 중 저장 안 하고 전원 끄면 날아가는 이유:
  작업 중인 내용 = RAM 에 있음
  저장(Save) = RAM → HDD/SSD 로 옮기는 행위

Spark / Kafka 도 마찬가지:
  Spark 캐시 → RAM 에 있음 → 재시작하면 사라짐
  Kafka 는 디스크에 저장 → 재시작해도 메시지 유지
```

---

---

# ③ OOM — Out Of Memory

```
책상(RAM) 이 좁은데 큰 백과사전(대용량 데이터) 을
한꺼번에 펼치려 할 때 발생

증상:
  프로그램이 갑자기 종료됨 (Killed)
  컴퓨터가 극도로 느려짐
```

## Swap — 가상 메모리

```
RAM 이 꽉 차면 OS 가 급한 대로 디스크 일부를 RAM 처럼 사용
→ 이걸 Swap (가상 메모리) 이라고 함

문제:
  디스크를 RAM 처럼 쓰는 것 → 속도 급감
  "나 책상이 좁아서 바닥에 책 놓고 일하느라 허리 끊어지겠어!"
```

## 확인 방법

```bash
# RAM 사용량 확인
free -h

# 실시간 모니터링
top
htop

# top 에서:
#   MiB Mem:  16000  total  2000  free  12000  used  2000  buff
#   MiB Swap:  2000  total  1800  free   200  used     ← Swap 쓰고 있으면 위험
```

```
서버 느려졌을 때 확인 순서:
  1. top / htop 실행
  2. Memory 사용량 확인
  3. RAM 99% + Swap 사용 중 → OOM 직전
  4. 해결: 코드 수정 (청크 처리) 또는 RAM 증설
```

---

---

# ④ DE 관점 — 실전 상황

## Spark OOM

```
Spark 는 RAM 에 데이터 올려서 처리
→ 데이터가 RAM 보다 크면 OOM 발생

Spark 의 대처:
  Spill to Disk — RAM 부족 시 자동으로 디스크 임시 사용
  속도는 느려지지만 작업은 계속됨

OOM 예방:
  partition 수 늘리기 (데이터를 더 잘게 쪼개기)
  spark.executor.memory 설정 늘리기
  불필요한 캐시 해제 (df.unpersist())
```

## Kafka vs Spark 메모리

```
Kafka:
  디스크에 저장 (기본 7일 보존)
  → 재시작해도 메시지 유지 ✅
  → RAM 에 의존 안 함

Spark:
  RAM 에서 처리
  → 재시작하면 캐시 사라짐
  → checkpointLocation 으로 처리 위치만 기억
```

---

---

# 한눈에 정리

|항목|RAM|HDD/SSD|
|---|---|---|
|속도|매우 빠름|느림|
|용량|수십 GB|수 TB|
|휘발성|전원 끄면 사라짐|영구 저장|
|역할|작업 공간 (책상)|보관 공간 (서고)|
|DE 관련|Spark In-Memory / OOM|Kafka 메시지 저장|