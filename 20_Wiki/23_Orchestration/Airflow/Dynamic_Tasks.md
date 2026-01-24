---
aliases:
  - Dynamic Task
  - 동적 생성
  - Loop Task
  - 반복문 태스크
tags:
  - Airflow
  - Python
  - Pattern
  - Automation
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Task_Dependencies]]"
  - "[[Dynamic_Task_Mapping]]"
---
## 개념 한 줄 요약

**Dynamic Task**는 태스크를 일일이 `task1 = ...`, `task2 = ...` 쓰지 않고, **Python의 `for`문**을 이용해 자동으로 찍어내는 기법이야.

---
## 왜 필요한가 (Why)

**문제점:**
- 만약 태스크를 100개 만들어야 한다면? 손으로 다 치다가 오타 나고 밤새워야 해. (노가다 🙅‍♂️)

**해결책:**
- `range(0, 100)`으로 반복문만 돌리면 3줄 만에 태스크 100개를 만들 수 있어. 
- 데이터 파티션(날짜별, 지역별) 처리를 할 때 필수야.

---
## Code Core Points

이 코드는 **"기차 놀이(Chaining)"** 패턴이야. 
태스크가 병렬로 도는 게 아니라, 앞 태스크가 끝나야 다음 태스크가 실행되게 연결했어.

```python
import pendulum
from airflow import DAG
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id='dynamic_task',
    start_date=pendulum.datetime(2026, 1, 20, tz="Asia/Seoul"),
    catchup=False,
) as dag:
    
    # 1. 태스크들을 담을 빈 리스트 생성
    tasks = []
    
    # 2. 0부터 4까지 5번 반복
    for i in range(0, 5):
        # 3. 태스크 생성 (ID: task0, task1, ... task4)
        task = EmptyOperator(task_id=f'task{i}')
        tasks.append(task)
        
        # 4. 의존성 연결 (기차 놀이 로직) 🚂
        # 첫 번째 놈(i=0)은 연결할 앞사람이 없으니까 패스!
        if i != 0:
            # 현재 태스크(i)는 이전 태스크(i-1)가 끝나야 실행됨
            tasks[i-1] >> tasks[i]
```

---
## Detailed Analysis (어떻게 연결된 거야?)

`tasks[i-1] >> tasks[i]` 이 부분이 마법의 코드야. 
루프를 돌면서 어떻게 변하는지 봐봐.

| **i 값** | **생성된 태스크** | **조건문 (i != 0)** | **실행되는 코드**            | **결과 (의존성)**       |
| ------- | ----------- | ---------------- | ---------------------- | ------------------ |
| **0**   | `task0`     | False (0이니까)     | (없음)                   | `task0` (시작)       |
| **1**   | `task1`     | True             | `tasks[0] >> tasks[1]` | `task0` -> `task1` |
| **2**   | `task2`     | True             | `tasks[1] >> tasks[2]` | `task1` -> `task2` |
| ...     | ...         | ...              | ...                    | ...                |
| **4**   | `task4`     | True             | `tasks[3] >> tasks[4]` | `task3` -> `task4` |

**결과 그래프:** `task0` ➡ `task1` ➡ `task2` ➡ `task3` ➡ `task4` 순서로 쭉 이어짐.

---
## Practical Context (병렬 vs 직렬)

**직렬(Sequential):**
- 앞뒤 관계가 중요할 때 사용. (예: 1월 데이터 처리 -> 2월 데이터 처리)

**병렬(Parallel):**
- 만약 `if`문 없이 그냥 `start >> task` 처럼 연결하면?
- 5개가 동시에 와르르 실행됨. (예: 5개 지역 데이터 동시 수집)

---
## 초보자가 자주 착각하는 포인트

1. "태스크 ID(`task_id`)가 겹쳐요!"
	- 반복문 돌릴 때 `task_id`를 똑같이 쓰면 에러 나.
	- 반드시 `f'task{i}'`처럼 변수(`i`)를 넣어서 **유니크한 이름**을 만들어줘야 해.

2. "리스트(`tasks`)는 꼭 필요한가요?"
	-  직렬연결(`>>`)을 하려면 **이전 태스크(`tasks[i-1]`)** 를 찾아야 하니까 리스트에 저장해두는 게 편해.
	- 병렬 연결이면 굳이 리스트 안 만들고 바로 연결해도 돼.

>  **Airflow 2.3+** 환경이라면 아까 배운 `for`문 방식보다는 이 **Dynamic Task Mapping**이 훨씬 우아하고 성능도 좋습니다.
> [[Dynamic_Task_Mapping]] 참고 