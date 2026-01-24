---
aliases:
  - TaskGroup
  - 태스크 그룹
  - SubDAG 대체
  - Airflow UI 정리
  - "@task_group"
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Python_Decorators]]"
---
## 개념 한 줄 요약

**Task Group**은 컴퓨터 바탕화면의 **"폴더"** 같은 거야.
지저분하게 널려 있는 태스크(파일)들을 논리적인 그룹(폴더)으로 묶어서, Airflow UI(Graph View)에서 클릭 한 번으로 펼쳤다 접었다 할 수 있게 해주는 기능이지.

---
## 왜 필요한가 (Why)

 **문제점 (Spaghetti DAG):** 
 - 태스크가 50개, 100개 넘어가면 선들이 엉켜서 도저히 흐름을 알아볼 수가 없어(Visual clutter)
 
 **해결책:** 
 - 관련된 태스크들(예: `전처리_1`, `전처리_2`, `전처리_3`)을 `[전처리_그룹]` 하나로 묶어두면 UI가 훨씬 심플해져.

----
##  Code Core Points

Task Group을 만드는 방법은 크게 두 가지야. (최신 방식인 데코레이터를 추천해!)

### A. 데코레이터 방식 (`@task_group`) - 추천! 

함수 위에 `@task_group` 모자만 씌우면 그 안에 있는 태스크들이 자동으로 묶여.

**데코레이터(`@task_group`)** 와 **엣지 라벨(`Label`)**, 그리고 **설정 상속(`retries`)** 까지 한 번에 이해하는 실전 코드야. 

```python
from datetime import datetime
from airflow import DAG
from airflow.decorators import task_group
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.utils.edgemodifier import Label

with DAG(
    dag_id='task_group',
    start_date=datetime(2023, 12, 1),
    schedule='@daily',
    # [Level 1] DAG 전체 기본값: 모든 태스크는 기본적으로 재시도 1회
    default_args={"retries": 1},
    catchup=False
) as dag:
    
    # 1. 태스크 그룹 정의 (함수 위에 모자 씌우기)
    # [Level 2] 그룹 설정: 이 그룹 안의 태스크들은 재시도 3회로 덮어씌움 (Override)
    @task_group(default_args={"retries": 3})
    def group1():
        """이 함수 설명(Docstring)은 Airflow UI에서 그룹에 마우스를 올리면 툴팁으로 떠요."""
        
        # task1: 별도 설정 없으므로 그룹 설정(Level 2)을 따라감 -> retries = 3
        task1 = EmptyOperator(task_id='task1')
        
        # task2: 자기만의 설정이 있음(Level 3) -> 그룹 설정 무시하고 내 설정 따름 -> retries = 2
        task2 = BashOperator(
            task_id="task2",
            bash_command="echo Hello World",
            retries=2
        )
        
        # 참고: 이 print는 DAG가 파싱(Parsing)될 때 스케줄러 로그에 찍힘 (실행 로그 아님)
        # print(task1.retries) -> 3 출력
        # print(task2.retries) -> 2 출력
        
    # 2. 그룹 밖의 태스크
    # task3: 그룹 밖이므로 DAG 기본값(Level 1)을 따라감 -> retries = 1
    task3 = EmptyOperator(task_id='task3')
    
    # 3. 의존성 설정 (Flow)
    # group1(전체)이 끝나면 -> "When completed" 라벨을 지나 -> task3 실행
    group1() >> Label("When completed") >> task3
```

![[스크린샷 2026-01-23 오후 12.50.13.png]]

1. **Retries 우선순위:**
    - `task1`: **3번** (그룹 설정 승리)
    - `task2`: **2번** (지조 있는 태스크 설정 승리)
    - `task3`: **1번** (DAG 기본 설정)

> 💡 **Tip:** `default_args` 상속 규칙이 헷갈린다면?
> 👉 [[Airflow_DAG_Skeleton#💡 Deep Dive `default_args`는 왜 쓸까? (상속의 마법)|기본 골격 문서의 "상속의 마법" 파트 참고하기]]

### B. 컨텍스트 매니저 방식 (`with TaskGroup`)

클래식한 방식이야. Operator들을 직접 묶을 때 써.

```python
from airflow.decorators import task_group

with TaskGroup("processing_group") as processing:
    t3 = BashOperator(...)
    t4 = BashOperator(...)
```

---
## Practical Context (실무 활용)

그룹을 만들 때 **"이 그룹은 어떤 조건일 때 실행되는지"** 화살표에 이름표(Label)를 붙여주면 가독성이 더 좋아져.

```python
from airflow.utils.edgemodifier import Label

# 화살표 위에 "When empty"라는 글씨가 뜸
my_task >> Label("When empty") >> other_task
```

---
## Detailed Analysis (SubDAG vs TaskGroup)

면접이나 실무에서 중요한 포인트!

**옛날 방식 (SubDAG):**
- 별도의 DAG를 또 만들어서 불러오는 방식.
- 버그가 많고(Deadlock), 설정이 복잡해서 **지금은 안 써(Deprecated).**

**요즘 방식 (TaskGroup):**
- 그냥 UI에서만 묶여 보이는 것뿐, 실제로는 하나의 DAG 안에 다 들어있어. 
- 성능 저하도 없고 설정도 쉬워. **무조건 이거 써!**