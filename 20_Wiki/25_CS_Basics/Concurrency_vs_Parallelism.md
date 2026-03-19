---
aliases:
  - 동시성과 병렬성
  - Concurrency
  - Parallelism
  - GIL
  - Multi-threading vs Multi-processing
tags:
  - CS
  - OS
  - Python
  - Performance
  - Airflow
related:
  - "[[Airflow_Queues_Scaling]]"
  - "[[CS_Blocking_vs_NonBlocking]]"
  - "[[Process_vs_Thread]]"
---
## 개념 한 줄 요약

**"혼자서 빨리 움직일 것인가, 둘이서 나눠서 할 것인가?"**

* **동시성 (Concurrency):** **요리사 1명**이 라면 끓이면서, 김밥 말고, 설거지하는 것. (동시에 하는 **척** 빠르게 번갈아 함)
* **병렬성 (Parallelism):** **요리사 2명**이 한 명은 라면 끓이고, 다른 한 명은 김밥 마는 것. (진짜 물리적으로 **동시에** 함)
---
## 왜 필요한가 (Why)

컴퓨터의 자원(CPU 코어)은 한정되어 있기 때문이야.

 **동시성 (Concurrency):**
 -  **I/O 작업(대기)** 이 많을 때, 기다리는 시간을 낭비하지 않으려고 씀. (효율성 극대화)
 
**병렬성 (Parallelism):**
- **CPU 작업(계산)** 이 너무 많을 때, 사람(코어)을 더 투입해서 빨리 끝내려고 씀. (속도 극대화)

---
## Practical Context (실무 활용)

이 비유 하나면 평생 안 까먹어.

### 상황 A: 동시성 (Concurrency) - IO Intensive 최적

* **환경:** 주방장 1명 (Single Core)
* **작업:** 라면 3개 끓이기 (물 끓는 시간 5분 대기 = I/O)
* **방식:**
    1. 냄비 1에 물 올림 (기다려야 함)
    2. 멍 때리지 않고 바로 냄비 2에 물 올림
    3. 바로 냄비 3에 물 올림
    4. 물 끓으면 면 넣음
* **결과:** 혼자서도 거의 동시에 3개를 끓여냄. **(Airflow `io_intensive` 큐의 원리)**

### 상황 B: 병렬성 (Parallelism) - CPU Intensive 최적

* **환경:** 주방장 3명 (Multi Core)
* **작업:** 양파 300개 다지기 (쉬지 않고 칼질 = CPU)
* **방식:**
    1. 주방장 A가 100개 다짐.
    2. 주방장 B가 100개 다짐.
    3. 주방장 C가 100개 다짐.
* **결과:** 셋이서 하니까 3배 빨라짐. **(Airflow `cpu_intensive` 큐의 원리)**

---
##  Code Core Points (Python의 함정: GIL)

파이썬에는 **GIL(Global Interpreter Lock)** 이라는 슬픈 제약이 있어. 
"한 번에 하나의 스레드만 파이썬 코드를 실행할 수 있다"는 규칙이야.

### A. Threading (동시성)

* **용도:** I/O 작업 (API 호출, DB 조회)
* **특징:** GIL 때문에 CPU 작업에 쓰면 오히려 느려짐. 하지만 I/O 대기 중에는 GIL을 풀어주기 때문에 효과적임.

```python
import threading

# 라면 물 올려놓고 딴짓하기 (I/O) -> 효과 좋음!
t1 = threading.Thread(target=io_task)
t2 = threading.Thread(target=io_task)
```

### B. Multiprocessing (병렬성)

- **용도:** CPU 작업 (데이터 연산, 머신러닝)
- **특징:** 아예 **별개의 프로세스(메모리 공간 분리)** 를 띄워서 GIL을 우회함. 진짜 병렬 처리가 가능하지만 메모리를 많이 먹음.

```python
import multiprocessing

# 양파 다지기 (CPU) -> 코어 수만큼 빨라짐!
p1 = multiprocessing.Process(target=cpu_task)
p2 = multiprocessing.Process(target=cpu_task)
```

---
## Detailed Analysis (Airflow와의 연결)

 Airflow 값들이 이제 이해될 거야.

1. **`io_intensive` 큐 (동시성 활용):**
    - 작업들이 대부분 `sleep`이나 `DB 대기`야.
    - 워커(CPU)가 1개라도 **Concurrency(동시 처리량)** 를 32, 64로 높게 잡아도 돼. (요리사 1명이 냄비 30개 관리 가능)

2. **`cpu_intensive` 큐 (병렬성 활용):**
    - 작업들이 미친 듯이 계산을 해.
    - 워커의 **Parallelism(병렬 처리량)** 을 물리적 CPU 코어 개수(예: 4개) 이상으로 잡으면 안 돼. (요리사 1명이 양손으로 칼질 4개를 동시에 못 함)

----
## 초보자가 자주 착각하는 포인트

1. "병렬성이 무조건 동시성보다 빠르다?"
	- 아니! 라면 1개 끓이는데 요리사 10명을 부르면(병렬성), 서로 부딪히고 대화하느라 더 느려져. 
	- (**Context Switching 오버헤드**). 작업 성격에 맞춰야 해.

2. "동시성은 동시에 하는 것이다?"
	- 엄밀히 말하면 **"동시에 하는 것처럼 보이게 빠르게 스위칭하는 것"** 이야. (Time Slicing).
	- 진짜 동시는 병렬성뿐이야

---
* **`sleep` 같은 I/O 작업** ➡ "요리사가 물 올려놓고 딴짓해도 되네?" ➡ **동시성(Concurrency)** 으로 해결 ➡ **`io_intensive` 큐** (한 놈이 많이 처리 가능). 
* **복잡한 연산 작업** ➡ "요리사가 칼질하느라 손을 못 떼네?" ➡ **병렬성(Parallelism)** 으로 해결 ➡ **`cpu_intensive` 큐** (사람 머릿수만큼만 처리 가능).