---
aliases:
  - 스택과 큐
  - Stack vs Queue
  - LIFO
  - FIFO
  - 자료구조
tags:
  - CS
  - DataStructure
  - Algorithm
  - Python
  - Airflow
related:
  - "[[Airflow_Queues_Scaling]]"
  - "[[Big_O_Notation]]"
---
## 개념 한 줄 요약

**스택(Stack)** 은 "프링글스 통"처럼 나중에 넣은 게 먼저 나오는 **후입선출(LIFO)** 구조이고, 
**큐(Queue)** 는 "맛집 대기 줄"처럼 먼저 온 사람이 먼저 들어가는 **선입선출(FIFO)** 구조야.

---
## 왜 필요한가 (Why)

데이터를 다룰 때 **"순서"** 를 어떻게 관리할지가 중요하기 때문이야.

* **스택 (LIFO):** "방금 한 거 취소해!" (Ctrl+Z) 처럼 **가장 최근 작업**이 중요할 때 필요해.
* **큐 (FIFO):** "접수된 순서대로 처리해!" (은행 창구, 프린터 인쇄) 처럼 **공정함과 순서 보장**이 중요할 때 필요해. (데이터 파이프라인의 핵심!)

---
## Practical Context (실무 활용)

###  스택 (Stack): "되돌리기와 추적"

1.  **함수 호출 (Stack Trace):** 에러 났을 때 `Traceback` 보면 맨 아래가 가장 최근에 호출된 함수지? 이게 스택 구조야.
2.  **웹 브라우저 '뒤로 가기':** 가장 최근에 본 페이지부터 꺼내서 보여주지.
3.  **DFS (깊이 우선 탐색):** 미로 찾기 할 때 막다른 길이면 바로 직전 갈림길로 돌아가는 원리.

###  큐 (Queue): "버퍼와 대기열"

1.  **Airflow Task Queue:** 스케줄러가 작업을 던지면 워커들이 **큐**에서 하나씩 꺼내서 실행해. 
	(먼저 스케줄 된 놈이 먼저 도는 게 원칙)
2.  **메시지 큐 (Kafka, RabbitMQ):** 데이터가 폭주할 때 잠시 보관해두는 완충지대.
3.  **BFS (너비 우선 탐색):** 나와 가까운 친구부터 순서대로 찾을 때.

---
## Code Core Points (Python)

파이썬에서는 `list`로 스택을 흉내 낼 수 있지만, **큐를 쓸 때는 반드시 `collections.deque`를 써야 해!** (성능 차이 때문)

### A. 스택 구현 (List 사용)

```python
stack = []

# 1. 넣기 (Push) -> append()
stack.append('A')
stack.append('B')

# 2. 꺼내기 (Pop) -> pop() : 맨 뒤(최근)에서 꺼냄
print(stack.pop()) # 'B' 출력 (LIFO)
```

### B. 큐 구현 (Deque 사용)

```python
from collections import deque

queue = deque()

# 1. 넣기 (Enqueue) -> append()
queue.append('A') # 손님 A 도착
queue.append('B') # 손님 B 도착

# 2. 꺼내기 (Dequeue) -> popleft() : 맨 앞(처음)에서 꺼냄
# 주의: 그냥 list의 pop(0)을 쓰면 데이터가 많을 때 엄청 느려짐!
print(queue.popleft()) # 'A' 출력 (FIFO)
```

---
## Detailed Analysis (성능 이슈)

왜 리스트로 큐를 만들면 안 될까?

 **List `pop(0)`:** 
- 맨 앞사람이 나가면, 뒷사람들이 한 칸씩 **다 앞으로 땡겨 앉아야 해.** 
- 데이터가 100만 개면 100만 번 이동이 일어나. (Time Complexity: $O(N)$)

**Deque `popleft()`:** 
- 그냥 "다음 사람!" 하고 포인터만 옮겨. 
- 아무리 데이터가 많아도 **즉시 처리**돼. (Time Complexity: $O(1)$)

---
## 초보자가 자주 착각하는 포인트

1. "Airflow는 무조건 FIFO인가요?"
	- 기본적으론 **큐(Queue)** 니까 FIFO가 맞아.
	- 하지만 `Priority Weight`(우선순위) 설정을 주면, 늦게 온 놈이 새치기해서 먼저 실행될 수도 있어. (VIP 줄 서기랑 똑같음!)

2. "스택은 쓸모없나요?"
	- 아니! 알고리즘 테스트(괄호 짝 맞추기 등)나 재귀 함수(Recursion) 이해할 때 필수야.

