---
aliases:
  - Blocking I/O
  - Non-Blocking
  - Sync vs Async
  - 동기 vs 비동기
  - 블로킹
tags:
  - CS
related:
  - "[[Airflow_Sensors]]"
  - "[[Airflow_Queues_Scaling]]"
  - "[[00_CS_HomePage]]"
  - "[[Python_DateTime]]"
---
# CS_Blocking_vs_NonBlocking

## 한 줄 요약

```
Blocking     작업이 끝날 때까지 기다림 (제어권 뺏김)
Non-Blocking 작업 시작만 하고 즉시 다른 일 (제어권 즉시 반환)
```

---

---

# ① 커피숍 진동벨 비유

```
Blocking:
  커피 주문하고 카운터 앞에 서서 하염없이 기다림
  → 뒷사람 주문 못 받음

Non-Blocking:
  주문하고 진동벨 받은 뒤 자리에서 폰 봄
  → 직원은 뒷사람 주문 계속 받음
```

---

---

# ② 왜 필요한가 — CPU 와 I/O 의 속도 차이

```
CPU      빛의 속도로 계산
I/O      CPU 에 비하면 거북이 (디스크 읽기 / 네트워크 응답)

문제:
  파일 읽기를 시켰는데 다 읽힐 때까지 CPU 가 멍 때리면 낭비

해결:
  "너 파일 읽는 동안 난 다른 거 처리하고 있을게" → Non-Blocking
```

---

---

# ③ Blocking vs Non-Blocking

## Blocking — 제어권 뺏김

```
함수를 호출하면 작업이 완전히 끝날 때까지 제어권 반환 안 함

예: requests.get() / time.sleep() / 일반 파일 읽기
특징: 순서대로 정직하게 실행 / 직관적이지만 대기 시간 낭비
```

```python
import time

def blocking():
    print("A: 주문")
    time.sleep(5)    # 5초 동안 완전히 멈춤
    print("A: 완료")
    print("B: 주문")  # A 끝날 때까지 B 못 시작
    time.sleep(5)
    print("B: 완료")
    # 총 소요: 10초
```

## Non-Blocking — 제어권 즉시 반환

```
함수 호출 후 "시작했어!" 하고 즉시 제어권 돌려줌
결과는 나중에 콜백(Callback) 또는 폴링(Polling) 으로 확인

예: asyncio / aiohttp
특징: 기다리는 시간에 다른 일 처리 → 전체 처리량 증가
```

```python
import asyncio

async def non_blocking():
    print("A: 주문 (진동벨 받음)")
    print("B: 주문 (진동벨 받음)")  # A 기다리는 동안 B 주문 가능

    await asyncio.sleep(5)   # 기다리는 동안 CPU 는 다른 일

    print("A, B: 동시에 완료")
    # 총 소요: 5초 (절반)
```

---

---

# ④ time.sleep 은 왜 I/O 작업인가?

```
CPU Bound:  수학 계산처럼 CPU 가 계속 일함
I/O Bound:  파일 읽기 / 네트워크처럼 CPU 가 대기
Sleep:      "10초 뒤에 깨워줘" → CPU 가 아무것도 안 함

→ sleep 은 진짜 I/O 는 아니지만
  CPU 를 안 쓰고 시간만 보내며 대기한다는 점에서
  I/O 작업과 동일한 패턴
```

```
OS 스케줄러 동작:
  1. sleep 호출 → OS 가 CPU 반납 (Running → Waiting)
  2. 비는 CPU 에 다른 프로세스 실행 (Context Switch)
  3. 시간 지나면 OS 가 다시 깨워서 실행
```

---

---

# ⑤ Airflow Sensor 적용

```
Sensor: "파일이 도착했는지 계속 확인하는 태스크"

mode='poke' (Blocking):
  워커가 자리 차지하며 계속 확인
  파일이 1시간 뒤에 오면 워커 1개가 1시간 낭비

mode='reschedule' (Non-Blocking):
  "파일 안 왔네? 10분 뒤에 다시 올게" → 자리 비움
  그 10분 동안 워커가 다른 DAG 처리
  → 효율적 ✅
```

---

---

# ⑥ 자주 하는 착각

```
"Non-Blocking 이 무조건 빠른가?"
  → 아님. 작업 하나 속도는 동일 (커피 나오는 시간은 5분으로 같음)
  → 전체 처리량(Throughput) 이 늘어나는 것
     1시간에 100잔 팔 거 → 200잔 팔 수 있게 됨

"Blocking = 동기(Sync), Non-Blocking = 비동기(Async) 같은 말?"
  → 엄밀히는 다르지만
  → 입문 단계에서는 동일하게 이해해도 무방
```

---

---

# 한눈에 정리

|항목|Blocking|Non-Blocking|
|---|---|---|
|제어권|작업 끝날 때까지 뺏김|즉시 반환|
|대기 중|아무것도 못 함|다른 작업 처리 가능|
|코드|`time.sleep()` / `requests.get()`|`asyncio.sleep()` / `aiohttp`|
|처리량|낮음|높음|
|복잡도|단순|복잡|
|Airflow|`mode='poke'`|`mode='reschedule'`|