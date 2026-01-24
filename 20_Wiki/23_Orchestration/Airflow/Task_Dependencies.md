---
aliases:
  - Task Dependency
  - 의존성
  - Bitshift Operator
  - 순서 정하기
  - ">>"
  - 리스트 의존성
tags:
  - Airflow
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Airflow_DAG_Operators]]"
---
## 개념 한 줄 요약

**의존성(Dependency)** 설정은 태스크들 사이의 **"선후 관계"** 를 정해주는 거야.
Airflow에서는 **`>>` (비트 시프트 연산자)** 를 사용해서 화살표 방향대로 실행 순서를 정해.

---
## 왜 필요한가 (Why)

**문제점:**
- 의존성을 설정 안 하면, 모든 태스크가 시작하자마자 동시에 와르르 실행돼버려.

**해결책:**
- "A가 끝나야 B를 한다"는 규칙을 정해줘야 논리적인 파이프라인이 됨.

---
## Core Syntax (핵심 문법)

### A. 기본 연결 (`>>`)

가장 많이 쓰는 방식. 화살표 방향으로 흐른다고 생각하면 돼.

```python
task1 >> task2        # task1 성공 후 task2 실행
task1 >> task2 >> t3  # 1 -> 2 -> 3 순서로 실행 (체이닝)
```

### B. 리스트 활용 (`[ ]`) 

여러 태스크를 **동시에(병렬로)** 실행하거나, 여러 태스크가 **다 끝날 때까지 기다릴 때** 리스트 `[]`를 써.

- **Fan-out (퍼뜨리기):** `task1 >> [t2, t3]`
    - task1이 끝나면 t2, t3가 동시에 시작됨.

- **Fan-in (모으기):** `[t2, t3] >> task4`
    - t2, t3가 **모두 끝나야** task4가 시작됨.

---
## Practical Context (코드 분석)

```python
# 1. task1이 끝나면 -> task2, 3, 4가 동시에 실행됨 (Fan-out)
# 2. task2, 3, 4가 모두 다 끝나면 -> task5가 실행됨 (Fan-in)

# normal way
# task1 >> task2
# task1 >> task3
# task1 >> task4
# task2 >> task5
# task3 >> task5
# task4 >> task5

# 리스트 활용
task1 >> [task2, task3, task4] >> task5
```

---
## 초보자가 자주 착각하는 포인트

1. **리스트끼리는 연결 불가:** `[t1, t2] >> [t3, t4]` ❌ (에러 발생!)
	- 리스트와 리스트를 바로 연결할 수는 없어.
	- 이럴 땐 `cross_downstream`이나 `chain`이라는 고급 기능을 써야 하는데, 일단은 안 된다고만 알아둬.

