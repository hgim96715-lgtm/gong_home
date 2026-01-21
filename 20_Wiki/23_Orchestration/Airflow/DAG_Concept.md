---
aliases:
  - DAG 개념
  - 방향성 비순환 그래프
  - Workflow
  - Task vs Instance
tags:
  - Airflow
  - 개념
  - 자료구조
  - 면접질문
related:
  - "[[Basic_DAG_Skeleton]]"
  - "[[DAG_Operators_Basic]]"
---
## 개념 한 줄 요약

**D**irected **A**cyclic **G**raph의 약자로, **"한쪽 방향으로만 흐르고(Directed), 절대 순환하지 않는(Acyclic) 작업의 흐름(Graph)"** 을 의미합니다.

---
## 왜 필요한가 (Why)

**문제 상황:**
작업 A가 B를 부르고, B가 다시 A를 부르는 구조(Loop)가 있다면, 컴퓨터는 영원히 그 작업을 끝내지 못하고 뺑뺑이를 돌게 됩니다. (무한 루프)

**해결책:**
Airflow는 DAG라는 수학적 구조를 강제함으로써 **"모든 작업은 반드시 끝이 있다"** 는 것을 보장합니다.
또한, 작업 간의 **선후 관계(Dependency)** 를 명확히 하여 "데이터가 준비되지 않았는데 분석을 시작하는" 실수를 방지합니다.

---
## 실무 맥락에서의 사용 이유

"데이터 파이프라인 짰어?"라고 묻는 것은 곧 **"DAG 그렸어?"** 라는 말과 같습니다.
실무에서는 "매일 아침 9시에(Schedule) 주문 데이터를 가져와서(Task A) 정제하고(Task B) DB에 넣는다(Task C)"는 하나의 업무 단위를 **DAG**라고 부릅니다.

---
## 상세 분석 (핵심 용어)

### 1. Node & Edge (구성 요소)

* **Node (Task):** 그래프의 점. 실행할 작업 그 자체입니다. (Python, Bash, SQL 등)
* **Edge (Dependency):** 그래프의 선. 작업의 실행 순서입니다. (`>>` 화살표로 표현)

### 2. Task vs Task Instance (가장 중요!)

이 둘을 구분하는 것이 Airflow 로그 분석의 핵심입니다.

| 구분 | Task (작업 정의) | Task Instance (작업 실체) |
| :--- | :--- | :--- |
| **비유** | '붕어빵 틀' | '오늘 구운 팥붕어빵' |
| **정의** | 코드에 적힌 클래스(`BashOperator`) 그 자체 | 특정 날짜(`execution_date`)에 실행된 Task |
| **상태** | 상태가 없음 (그냥 코드임) | Success, Failed, Running, Retry 등의 상태를 가짐 |

---
## 초보자가 자주 착각하는 포인트

1.  **"DAG 안에 루프(Loop)를 넣고 싶은데요?"**
    * 절대 불가능합니다. Airflow 철학에 위배됩니다. 반복문이 필요하면 DAG 안에서 Python 코드로 for문을 돌리거나, `Dynamic DAG`라는 고급 기술을 써야 합니다.

2.  **"Task Instance가 실패하면 Task가 실패한 건가요?"**
    * 엄밀히 말하면 "2026년 1월 21일 자 Task Instance"가 실패한 것입니다. 코드는 그대로여도 내일 자 Instance는 성공할 수도 있습니다. (데이터 문제일 수 있으니까요.)