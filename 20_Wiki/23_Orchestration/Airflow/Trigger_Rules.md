---
aliases:
  - Trigger Rule
  - 트리거 룰
  - 의존성 규칙
  - one_success
  - all_done
tags:
  - Airflow
  - Dependency
  - Logic
  - FlowControl
related:
  - "[[Task_Dependencies]]"
  - "[[Branching]]"
  - "[[Airflow_DAG_Skeleton]]"
---
## 개념 한 줄 요약

**Trigger Rule**은 태스크가 실행되기 위한 **"입장 조건"** 이야.
"내 앞의 태스크(Upstream)들이 모두 성공해야만 나도 시작할 거야!"(기본값)라는 조건을, "하나만 성공해도 할래" 또는 "다 망해도 할래"처럼 바꿔주는 설정이지.

---
## 왜 필요한가 (Why)

**문제점:**
- Airflow의 기본 규칙은 **`all_success`** 야. 
- 즉, 내 앞의 부모님이 10명인데 그중 한 명이라도 **실패(Failed)** 하거나 **스킵(Skipped)** 되면, 나도 덩달아 실행을 안 해버려.

**해결책:**
- "A랑 B 중에 **하나만 성공해도** 다음 단계로 넘어가고 싶어." (`one_success`)
- "성공하든 실패하든 **무조건 뒷정리(Cleanup)** 는 해야 해." (`all_done`)
- "브랜치에서 **스킵된 애가 있어도** 나는 실행돼야 해." (`none_failed_min_one_success`)

---
##  Code Core Points

 **"둘 중 하나만 성공해도 실행되는"**  one_success 패턴 확인 

```python
import pendulum
from airflow import DAG
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id='trigger_rule',
    start_date=pendulum.datetime(2023, 12, 1, tz="Asia/Seoul"),
    catchup=False,
) as dag:

    # 부모 태스크 2개
    task1 = EmptyOperator(task_id='task1')
    task2 = EmptyOperator(task_id='task2')
    
    # 자식 태스크 (여기가 핵심! ✨)
    # task1이나 task2 둘 중 하나만 성공해도 task3는 실행됨
    task3 = EmptyOperator(
        task_id='task3', 
        trigger_rule='one_success' 
    )

    # 병렬 구조
    [task1, task2] >> task3
```

![[스크린샷 2026-01-23 오후 3.24.08.png|400x200]]

>pendulum 에 대해 알고싶다면? => [[Airflow_DAG_Skeleton#`start_date` 언제부터 시작할까? (ft. Pendulum,Timezone)|pendulm]] 참조 
---
## Detailed Analysis (옵션 총정리)

실무에서 **자주 쓰는 4대장**만 외우면 돼.

|**룰 이름**|**설명**|**언제 쓸까? (Use Case)**|
|---|---|---|
|**all_success** (기본값)|앞 태스크가 **모두 성공**해야 함.|일반적인 파이프라인. (가장 안전함)|
|**all_done**|성공/실패/스킵 상관없이 **다 끝나기만 하면** 함.|**전송 실패 알림 보내기**, 클러스터 종료하기 등 **무조건 실행**되어야 할 때.|
|**one_success**|앞 태스크 중 **하나라도 성공**하면 바로 시작.|빠른 처리가 필요할 때 (예: 여러 서버에서 데이터 다운로드 시도 중 하나만 되면 됨).|
|**none_failed**|실패한 놈만 없으면 됨 (**스킵은 봐줌**).|⭐️ **Branching** 다음에 합쳐지는 태스크에 필수! (이거 안 쓰면 뒤에 다 스킵됨)|

---
## Practical Context (Branching과 찰떡궁합)

**[[Branching]]** 을 배우고 나서 이걸 배우는 이유가 있어.
브랜치를 타면 **선택받지 못한 길은 `Skipped`** 가 되잖아?

- 만약 뒤에 붙은 태스크가 기본값(`all_success`)이면? ➡ "어? 앞놈이 스킵됐네? 나도 안 해!" 하고 같이 죽어버려.
- 그래서 브랜치 뒤에 태스크를 붙일 때는 보통 **`none_failed`** 나 **`none_failed_min_one_success`** 를 써야 해.

---
## 초보자가 자주 착각하는 포인트

1. "Skipped도 Success 아닌가요?"
	- 아니! Airflow에서 **Success(초록)** 와 **Skipped(분홍)** 는 엄연히 달라.
	- `all_success`는 **Skipped**를 만나면 조건 불충족으로 판단해서 실행 안 해.

2. "`one_success`는 나머지 기다리나요?"
	- 아니! `task1`이 성공하자마자 `task2`가 아직 돌고 있어도 `task3`는 **즉시 시작**해버려. (성격 급한 녀석임)

>**Obsidian 연결:** 나중에 [[Branching]] 코드를 짤 때 "어? 뒤에 태스크가 왜 안 돌지?" 싶으면 무조건 이 문서를 다시 보게 될 거야.