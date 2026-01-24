---
aliases:
  - Params
  - 파라미터
  - Runtime Config
  - Jinja Template
  - DAG Config
tags:
  - Airflow
  - Configuration
  - Jinja
  - BashOperator
related:
  - "[[Airflow_DAG_Skeleton]]"
  - "[[Airflow_XComs]]"
---
## 개념 한 줄 요약

**Params(파라미터)** 는 DAG를 실행할 때 넣어주는 **"입력 변수(Argument)"** 야.
코드를 수정하지 않고도, Airflow 웹 화면에서 "재생 버튼(Trigger)"을 누를 때 값을 바꿔서 실행할 수 있게 해주는 기능이지.

## 왜 필요한가 (Why)

* **문제점:** 코드를 `bash_command='echo v1'`이라고 짜면, 값을 `v2`로 바꾸고 싶을 때마다 코드를 고치고 배포해야 해. (Hardcoding)
* **해결책:** `{{ params.p1 }}`처럼 구멍을 뚫어놓고, 실행할 때 원하는 값을 채워 넣는 거야. "이번엔 v1으로 돌려!", "이번엔 v2로 돌려!" 하고 명령만 내리면 됨.

---
## Practical Context (Trigger DAG w/ Config)

실무에서는 **"수동 실행(Manual Trigger)"** 할 때 진짜 빛을 발해.

1. Airflow UI 우측 상단의 **[▶️ 재생 버튼]** 클릭 -> **[Trigger DAG w/ config]** 선택.
2.  **Run Parameters** 화면이 뜨는데, 우리가 코드에 적어둔 `p1`, `p2`가 입력창으로 딱 떠 있어!
3. 여기서 `v1`을 지우고 **`new_value`** 라고 적은 뒤 **[트리거]** 버튼 클릭.
4. 결과: 코드는 `v1`이지만, 이번 실행에서만 **`new_value`** 로 바뀌어서 돌아가!
    - **활용 예:** "2023년 1월 데이터만 다시 돌려줘", "모델 버전을 v2 말고 v3로 테스트해 봐."

> **💡 Tip:** 만약 옛날처럼 JSON으로 입력하고 싶다면 하단의 **[고급 옵션(Advanced Options)]** 을 누르면 돼.
---
##  Code Core Points

이 코드는 **스코프(Scope, 적용 범위)** 를 이해하는 데 완벽해.

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
import pendulum

# 1. DAG 레벨 파라미터 (전역 변수)
# 이 DAG 안의 모든 태스크가 기본적으로 가져다 쓸 수 있음
params = {
    "p1": "v1",
    "p2": "v2",
}

with DAG(
    dag_id='params_argument',
    start_date=pendulum.datetime(2024, 6, 20, tz="Asia/Seoul"),
    params=params, # 여기서 등록!
    catchup=False 
) as dag:
    
    # task1: DAG의 기본 params를 그대로 사용
    task1 = BashOperator(
        task_id='task1',
        # Jinja 템플릿 문법 {{ }}을 써서 값을 꺼내옴
        bash_command='echo {{ params.p1 }}'  # 결과: echo v1
    )
    
    # task2: 자기만의 params를 추가/덮어쓰기 (지역 변수)
    task2 = BashOperator(
        task_id='task2',
        # p2는 DAG꺼(v2), p3는 내꺼(v3) 사용
        bash_command='echo {{ params.p2 }} {{ params.p3 }}', # 결과: echo v2 v3
        params={"p3": "v3"} # Task 레벨에서 추가 정의
    )
    
    task1 >> task2
```

---
## Detailed Analysis (Jinja Template)

여기서 `{{ params.p1 }}` 같은 문법이 보이지? 이걸 **Jinja 템플릿**이라고 해.

- **원리:** Airflow가 태스크를 실행하기 직전에 `{{ }}` 안의 내용을 실제 값(`v1`)으로 **"치환(Render)"** 해주는 거야.
- **주의:** 모든 파라미터에서 쓸 수 있는 건 아니고, `bash_command`처럼 **"Templated Field"** 라고 지정된 곳에서만 작동해. (BashOperator는 웬만하면 다 됨)

---
## 초보자가 자주 착각하는 포인트

1. "파이썬 변수(`f-string`)랑 다른가요?"
	- 완전 달라! `f"{params['p1']}"`는 코드가 **파싱될 때(DAG 로딩 시점)** 값이 고정돼버려.
	- `{{ params.p1 }}`는 **실행될 때(Runtime)** 값이 결정돼. 그래서 웹 화면에서 입력한 값을 받으려면 무조건 `{{ }}`를 써야 해.

2. "XCom이랑 뭐가 달라요?" 
	- **Params:** (엄마가 준 용돈) 시작할 때 쥐어주는 **입력값**.
	- **XCom:** (내가 번 돈) 태스크가 일하고 남긴 **결과값**.

>이 `{{ }}` 문법에 익숙해지면 나중에 `{{ ds }}` (오늘 날짜) 같은 **Airflow 매크로**도 자유자재로 쓸 수 있어. 
>고수들은 미리 `params`로 구멍을 뚫어놓고, 웹 UI에서 JSON 설정(`{"target_date": "2024-01-01"}`)만 쓱 바꿔서 실행하죠. 이 기능을 잘 쓰면 **"재배포 없이 유연한 파이프라인"** 을 만들 수 있습니다! 

