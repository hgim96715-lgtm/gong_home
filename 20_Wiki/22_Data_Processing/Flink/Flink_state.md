---
aliases:
  - Stateful
  - 상태관리
  - Keyed State
tags:
  - PyFlink
related:
  - "[[Spark_Streaming_Stateful_Stateless|Stateful vs Stateless]]"
  - "[[PyFlink_KeyBy_DeepDive]]"
  - "[[00_Apache Flink_HomePage]]"
---
## 한줄 요약

**"이벤트가 지나가도 사라지지 않고 남아있는 '기억(Memory)'이다."** 
단순히 들어오는 데이터를 하나씩 처리하고 잊어버리는 것이 아니라, **여러 이벤트에 걸쳐 정보를 유지**하는 기술입니다.

---
## 왜 필요한가? (Why)

**문제점:**

- 데이터가 들어오는 족족 처리하고 잊어버린다면(Stateless), "지난 1시간 동안의 합계"나 "A유저의 누적 클릭 수" 같은 걸 구할 수 없습니다.    
- 과거의 데이터와 현재의 데이터를 연결할 방법이 없습니다.

**해결책:**

- **State**를 사용하면 윈도우 집계(Windowed Aggregates), 이벤트 순서 감지, 머신러닝 모델 파라미터 업데이트, 과거 데이터 관리 등을 할 수 있습니다.
- Flink는 이 상태를 추적하여 **장애가 발생해도 복구(Fault Tolerance)** 할 수 있게 해주고, 병렬 인스턴스 간에 **투명한 크기 조정(Rescaling)** 을 가능하게 합니다.

---
## 실전 맥락 (Practical Context)

데이터 엔지니어링 실무에서는 크게 두 가지 변환(Transformation) 방식으로 나뉩니다.

### 1. Stateless Transformations (건망증)

- **특징:** 각 레코드를 독립적으로 처리합니다. 과거의 입력값을 기억하지 않습니다.
- **장점:** 구조가 단순하고 확장이 매우 쉽습니다.
- **함수 예시:** `map()`, `flatMap()`, `filter()`.
- **사용 사례:** 문자열 대문자 변환, Null 값 제거, 필드 쪼개기(Explode).

### 2. Stateful Transformations (기억력 천재)

- **특징:** 이전에 보았던 요소들에 의존하여 결과를 계산하기 위해 **내부 상태(Internal State)** 를 유지합니다.
- **장점:** Flink가 이 상태를 안전하게 저장하므로 장애 발생 시 복구할 수 있습니다.
- **함수 예시:** `keyBy()` + `sum()`/`reduce()`, `window()`.
- **사용 사례:** 단어 수 세기(Word Count), 세션 윈도우, 순서 감지, 클릭스트림 조인.

---
## Code Core Points: State 조작 공식

Flink State는 일반 변수처럼 `a = 10`이라고 쓸 수 없습니다. 
반드시 **전용 메서드**를 써야 합니다.

**[code]**

```python
# 1. 읽기 (Read)
current_count = self.my_state.value()

# 2. 수정 (Modify)
if current_count is None:
    current_count = 0
new_count = current_count + 1

# 3. 저장 (Update)
self.my_state.update(new_count)

# 4. 삭제 (Clear - 특정 조건 시)
if new_count >= 100:
    self.my_state.clear()
```

### `.value()` : 금고 들여다보기

- **역할:** 현재 이 키(Key)에 저장된 상태값을 가져옵니다.
- **주의:** 데이터가 처음 들어온 상태라면 `None`을 반환합니다. 
- 따라서 **반드시 `is None` 체크**를 해서 초기값을 잡아줘야 합니다.

### `.update(value)` : 금고에 넣고 문 잠그기

- **역할:** 새로운 값을 상태 저장소에 물리적으로 기록합니다.
- **핵심:** 이 메서드를 호출하는 순간, Flink는 "아, 이 데이터는 나중에 서버 터져도 내가 책임지고 복구해줄게!"라고 약속하는 것과 같습니다.

### `.clear()` : 금고 비우기

- **역할:** 해당 키의 상태를 완전히 삭제합니다.
- **Why?** 데이터 엔지니어링에서 **메모리 관리**는 생명입니다. 
- 1년 전 유저의 데이터를 계속 들고 있을 필요가 없다면 `clear()`로 메모리를 확보해야 합니다. (OOM 에러 방지)



---
## 상세 분석: Keyed State (키별 상태)

Flink에서 가장 많이 쓰이는 상태 관리 방식입니다.

> **"특정 Key에 대한 데이터는 특정 사물함(State)에만 보관한다."**


**작동 원리:**

- 이벤트의 **Key**와 상태가 묶여 있습니다(Tied). 분산된 Key-Value 저장소에 저장됩니다.
- 반드시 `keyBy()`를 통해 **Keyed Stream**이 된 상태에서만 작동합니다.
- 각 키의 상태는 해당 키를 처리하는 **Operator 인스턴스(Task)에 로컬(Local)** 로 저장됩니다.

**데이터 흐름 예시 (Diagram 설명):**
1. **Source:** 데이터가 무작위로 들어옵니다 (Key: A, B, Y, D, E, Z).
2. **Redistribution (`keyBy()`):** 데이터를 키별로 헤쳐 모이게 합니다.
3. **Stateful Operator:**
    - `Task 1`: **A, B, Y** 키만 담당하며, 이들의 상태값만 관리합니다
    - `Task 2`: **D, E, Z** 키만 담당하며, 이들의 상태값만 관리합니다.

- 이렇게 키 그룹(Key Groups)을 통해 파티셔닝 되므로, 나중에 작업(Task) 수를 늘리거나 줄일 때(Scaling) 동적으로 재분배가 가능합니다.

---
## Common Beginner Misconceptions (초보자의 실수)

1. **"모든 변수(`variable`)가 State 아닌가요?"**
    - **오해:** 파이썬 클래스 내부의 일반 변수(`self.count = 0`)도 상태라고 생각합니다.
    - **현실:** 일반 변수는 Flink가 관리해주지 않습니다. 서버가 재시작되면 `0`으로 초기화되어 데이터가 날아갑니다. 
    - Flink의 `ValueState`, `ListState` 등을 써야 **장애 복구**가 됩니다.

2. **"keyBy 없이 State를 쓸 수 있나요?"**
    - **오해:** 그냥 `map` 함수 안에서 상태를 저장하고 싶어 합니다.
    - **현실:** 가능은 하지만(`Operator State`), 대부분의 비즈니스 로직(유저별 합계 등)은 `Keyed State`가 필요하므로 `keyBy()`가 선행되어야 합니다.