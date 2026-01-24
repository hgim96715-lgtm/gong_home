---
aliases:
  - Dynamic Task Mapping
  - expand
  - partial
  - map
  - 동적 매핑
tags:
  - Airflow
  - TaskFlow
  - Optimization
  - Modern
related:
  - "[[Dynamic_Tasks]]"
  - "[[Python_Decorators]]"
---
## 개념 한 줄 요약

**Dynamic Task Mapping**은 Airflow에게 "이 리스트에 있는 개수만큼 알아서 복제해!"라고 명령하는 **최신 기술(Airflow 2.3+)** 이야.
Python의 `map()` 함수처럼, 리스트의 각 요소에 똑같은 작업을 적용하는 **Map-Reduce** 패턴을 아주 쉽게 구현할 수 있어.

---
## 왜 필요한가 (Why)

**기존 방식(`for`문)의 한계:**
-   DAG가 **파싱(Parsing)** 될 때 개수가 정해져야 해. (실행 도중에 데이터가 몇 개 들어올지 모르면 못 씀)
- 태스크가 1,000개면 그래프 그리느라 웹 서버가 버벅거려.

**해결책 (Mapping):**
- **Lazy Loading:** 실행되는 순간(Runtime)에 데이터 개수를 세서 태스크를 늘려.
- **UI 최적화:** 1,000개가 생겨도 `[ ]` 괄호 하나로 묶여서 보여줘서 깔끔해.

---
## Code Core Points

핵심 키워드는 딱 두 개야. 
**`expand` (늘리기)** 와 **`partial` (고정하기)**.

```python
import pendulum
from airflow import DAG
from airflow.decorators import task

with DAG(
    dag_id='dynamic_task_mapping',
    start_date=pendulum.datetime(2026, 1, 20, tz="Asia/Seoul"),
    catchup=False
) as dag:

    # 1. 리스트를 만들어내는 태스크 (Producer)
    @task
    def get_filenames():
        # 실제로는 DB나 API에서 가져오겠지? 지금은 3개만 리턴.
        return ["file_a.csv", "file_b.csv", "file_c.csv"]

    # 2. 리스트를 받아서 처리할 태스크 (Consumer)
    @task
    def process_file(filename, folder_path):
        print(f"Processing {filename} in {folder_path}")

    # --- 여기가 마법이 일어나는 곳! ✨ ---
    
    file_list = get_filenames()
    
    # partial: 모든 태스크에 '공통'으로 들어갈 고정값 (folder_path)
    # expand: 리스트 개수만큼 '쪼개져서' 들어갈 변동값 (filename)
    process_file.partial(folder_path="/data/input").expand(filename=file_list)
```

---
## Detailed Analysis (`expand` vs `partial`)

이 두 가지를 구분하는 게 핵심이야.

|**메서드**|**의미**|**비유**|**역할**|
|---|---|---|---|
|**`.partial()`**|**부분 고정**|"붕어빵 틀"|모든 붕어빵에 똑같이 들어가는 재료 (밀가루 반죽)|
|**`.expand()`**|**확장(매핑)**|"내용물 주입"|팥, 슈크림, 피자... 종류별로 붕어빵을 찍어내는 변수|

**코드 해석:** 
- "모든 태스크의 `folder_path`는 `/data/input`으로 고정하고(`partial`),
- `filename`은 `file_list`에 있는 개수(3개)만큼 태스크를 복제해서(`expand`) 넣어라!"

---
## Practical Context (UI의 변화)

**Graph View:**
- 기존 `for`문 방식은 태스크가 `task0`, `task1`, `task2`... 이렇게 옆으로 줄줄이 늘어섰지?

**Mapping:**
- 딱 하나의 태스크 박스만 보이고, 이름 옆에 **`[ ]`** 표시가 생겨.
	- 이걸 클릭하면 그 안에 `Mapped Task Index: 0`, `1`, `2`... 이렇게 숨어 있어.
	- 1만 개를 돌려도 UI가 깔끔하게 유지되는 비결이야!

---
## 초보자가 자주 착각하는 포인트

1. **"리스트 말고 딕셔너리는 안 되나요?"**
	- 돼! `expand_kwargs`라는 걸 쓰면 되는데, 일단 리스트(`expand`)부터 마스터하고 넘어가자.

2. "XCom 안 쓰고 그냥 리스트 넣어도 되나요?"
	- 응! `expand(filename=['a', 'b'])` 처럼 직접 리스트를 박아도 동작해.
	- 하지만 보통은 앞 태스크의 결과(XCom)를 받아서 동적으로 처리할 때 진가를 발휘하지.

>이 방식은 [[Dynamic_Tasks]] (for문 방식)의 **상위 호환** 기술이야. 
>Airflow 버전이 2.3 이상이면 무조건 이걸 쓰는 걸 추천해!