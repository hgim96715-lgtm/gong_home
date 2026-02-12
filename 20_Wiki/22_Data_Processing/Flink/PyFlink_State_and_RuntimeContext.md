---
aliases:
  - RuntimeContext
  - ValueStateDescriptor
  - Flink State 문법
tags:
  - PyFlink
related:
  - "[[Flink_state]]"
  - "[[PyFlink_Import_Analysis]]"
  - "[[PyFlink_KeyedProcessFunction]]"
  - "[[Python_Generators_Yield]]"
---
## Concept Summary

**"Flink라는 은행(RuntimeContext)에 가서, 신청서(Descriptor)를 내고, 내 전용 금고(State)를 배정받는 과정."** 
단순히 변수(`self.count = 0`)를 쓰는 게 아니라, **Flink가 관리하는 영구적인 저장소**를 얻어내는 공식 절차입니다.

---
## 왜 필요한가? (Why)

**문제점 (일반 파이썬 변수):**

- `self.count = 0` 이라고 쓰면, 서버가 고장 나서 재부팅될 때 **0으로 초기화**되어 데이터가 다 날아갑니다.

**해결책 (Flink State):**

- **RuntimeContext**를 통해 만든 State는 하드디스크(Checkpoint)에 안전하게 저장됩니다.
- 서버가 터져도 **"마지막 기억(State)"** 을 되살려줍니다. 
- 그러려면 Flink한테 "나 이거 저장할 거야!"라고 정식으로 등록해야 합니다.

---
## 핵심 공식: 3단 합체 (The Formula) ⭐️

이 코드는 무조건 **`open()` 함수** 안에서만 작성해야 합니다.

```python
# [공식]
# 내_변수 = (은행원).get_state( 신청서("이름", 타입) )

self.my_state = self.get_runtime_context().get_state( 
    ValueStateDescriptor("my_state_name", Types.INT()) 
)
```

- 1단계: 은행원 소환 (`get_runtime_context`)
	- **역할:** Flink 시스템과 대화하는 창구입니다.
	- **코드:** `self.get_runtime_context()`
	- **의미:** "저기요, Flink 시스템 관리자님!"

- 2단계: 신청서 작성 (`ValueStateDescriptor`)
	- **역할:** 어떤 State를 만들지 적어내는 서류입니다.
	- **코드:** `ValueStateDescriptor("statue_name", Types.FLOAT())`
	- 필수 항목:
		- **이름(Name):** 상태의 ID. (예: "sum", "average", "last_login")
		-  **타입(Type):** 저장할 데이터 종류. (`Types.STRING()`, `Types.INT()` 등)

- 3단계: 금고 배정 (`get_state`)
	- **역할:** 신청서를 내고 실제 사용할 수 있는 객체(Handle)를 받습니다.
	- **코드:** `.get_state(...)`

---
## 실전 코드 패턴 (Code Skeleton)

```python
from pyflink.datastream.functions import KeyedProcessFunction, RuntimeContext
from pyflink.common.state import ValueStateDescriptor
from pyflink.common import Types

class MyProcessFunction(KeyedProcessFunction):
    
    # 1. 변수 선언 (껍데기만)
    def __init__(self):
        self.sum_state = None

    # 2. 초기화 (실제 State 배정) - 여기가 핵심!
    def open(self, runtime_context: RuntimeContext):
        # 3단 합체 공식 적용
        descriptor = ValueStateDescriptor("sum", Types.FLOAT())
        self.sum_state = runtime_context.get_state(descriptor)

    # 3. 사용 (데이터 처리)
    def process_element(self, value, ctx):
        # State 읽기 (.value)
        current_sum = self.sum_state.value()
        
        # Null 체크 (처음엔 비어있으니까)
        if current_sum is None:
            current_sum = 0.0
            
        # 로직 수행
        current_sum += value
        
        # State 저장 (.update)
        self.sum_state.update(current_sum)
```

---
## Common Beginner Misconceptions (초보자 실수)

### ① "open() 말고 `__init__`에 쓰면 안 되나요?"

- **안 됩니다!** ❌
- `__init__`은 파이썬 객체가 생성될 때 실행되는데, 이때는 아직 Flink 서버(Runtime)와 연결되기 전입니다.
- **반드시 `open()`** 에서 해야 "은행 문이 열리고" 나서 신청서를 낼 수 있습니다.

### ② "이름("sum")을 아무거나 써도 되나요?"

- 네, 하지만 **한 함수 안에서 중복되면 안 됩니다.**
- `sum_state`도 "data", `count_state`도 "data"라고 이름 지으면, Flink는 같은 금고라고 생각해서 데이터를 덮어써 버립니다.

### ③ "State 값을 그냥 `self.sum_state = 10` 하면 안 되나요?"

- **절대 안 됩니다!** ❌
- `self.sum_state`는 금고(객체) 자체입니다.
- 값을 넣으려면 **`.update(10)`**, 값을 꺼내려면 **`.value()`** 메서드를 써야 합니다.