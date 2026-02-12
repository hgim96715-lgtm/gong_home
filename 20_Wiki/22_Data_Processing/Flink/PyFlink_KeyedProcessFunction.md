---
aliases:
  - process_element
  - KeyedProcessFunction구조
  - PyFlink API
  - on_timer
  - 타이머제어
tags:
  - PyFlink
related:
  - "[[PyFlink_State_and_RuntimeContext]]"
  - "[[Flink_state]]"
---
## Concept Summary(한줄 요약)

**"Flink가 데이터를 받을 때마다 실행하기로 약속된 '이벤트 리스너(Event Listener)'이다."** 
`KeyedProcessFunction`을 상속받아 클래스를 만들 때 반드시 정의해야 하는 추상 메서드입니다.

**핵심 차이점 (Java vs PyFlink):**
- **Java:** `out.collect()`를 사용하여 접시에 담는 방식.
- **PyFlink:** **`yield`** 키워드를 사용하여 데이터를 하나씩 내보내는 **Generator** 방식.

---
## process_element: 재료 모으고 알람 맞추는 곳

### Code Core Points: 실행 흐름

1. 데이터 `A` 도착 → Flink가 `process_element(A, ...)` 호출.
2. 상태(State) 업데이트 (누적 합계 등).
3. 타이머 설정 (필요시).
4. 결과 배출이 필요하면 **`yield`** 사용.

### 파라미터 해부 (PyFlink 전용)

```python
# out 파라미터가 없습니다! 리턴값(yield)으로 내보냅니다.
def process_element(self, value, ctx: KeyedProcessFunction.Context):
```

- **`value` (재료):** 현재 스트림에 들어온 **데이터 한 건** (튜플 등).
- **`ctx` (도구):** 시스템 정보 접근 (타이머 설정, 현재 Key 확인 등).
	- `ctx.timer_service()`: 알람 예약
	- `ctx.get_current_key()`: 현재 처리 중인 Key 확인.
	- `ctx.timestamp()`: 이벤트 시간 확인.

---
## [상세] TimerService와 시간 확인법

`ctx.timer_service()`를 통해 현재 시간을 확인하고 알람을 예약합니다.

- **`current_processing_time()`**: 서버(Task Manager)의 **현재 시각 (ms)**.
- **`register_processing_time_timer(시간)`**: 해당 시간에 `on_timer`가 실행되도록 예약.

**타이머 설정 공식**

```python
# 1. 알람 울릴 시간 계산 (현재 서버 시간 + 10초)
# 단위는 반드시 밀리초(ms)입니다! (+10000)
target_time = ctx.timer_service().current_processing_time() + 10000

# 2. 알람 등록
ctx.timer_service().register_processing_time_timer(target_time)

# 3. 예약 정보 저장 (중복 예약 방지용 State)
self.timer_state.update(target_time)
```

---
## Detailed Analysis: 한 줄씩 뜯어보기 (`process_element`)

```python
# PyFlink: "데이터 왔다! value랑 ctx만 줄 테니, 결과는 yield로 줘!"
def process_element(self, value, ctx: KeyedProcessFunction.Context):
    
    # 1. 상태에서 기존 값을 꺼내옴
    current_sum = self.sum_state.value()
    
    # 2. 값 누적 (None 체크 필수)
    if current_sum is None:
        current_sum = 0.0
    current_sum += value[1] 
    
    # 3. 상태 업데이트 (Update)
    self.sum_state.update(current_sum)
    
    # 4. (필요시) 즉시 결과 내보내기
    # yield f"현재 누적값: {current_sum}"
```

---
## [완성] on_timer 함수: 알람이 울릴 때의 로직

예약한 시간이 되었을 때 Flink가 호출하는 함수입니다. 
**여기서도 `out` 객체는 없으며, `yield`를 사용합니다.**

### Code Core Points

1. **기억 소환:** `.value()`로 State에서 데이터 꺼내기.
2. **계산:** 평균 구하기 등 비즈니스 로직 수행.
3. **전송:** **`yield 결과값`** 으로 다음 단계로 데이터 토스.
4. **청소:** `.clear()`로 상태 초기화.

### 파라미터 해부 (PyFlink 전용)

```python
def on_timer(self, timestamp: int, ctx: KeyedProcessFunction.OnTimerContext):
```

- **`timestamp`:** 알람이 울린 시각 (ms).
- **`ctx`:** 타이머 실행 시점의 문맥 정보. (`ctx.get_current_key()`: 현재 처리 중인 Key 확인. , 사용 가능)

---
## Detailed Analysis: 한 줄씩 뜯어보기 (`on_timer`)

```python
# PyFlink: "따르릉! 시간 됐다. 결과 있으면 yield로 던져!"
def on_timer(self, timestamp: int, ctx: KeyedProcessFunction.OnTimerContext):
    
    # 1. 재료 꺼내기
    current_sum = self.sum_state.value()
    current_count = self.count_state.value()
    
    # 2. 결과 계산
    if current_count is not None and current_count > 0:
        avg_temp = current_sum / current_count
        state_key = ctx.get_current_key()
        
        # 3. 결과 배출 (Generator 방식) ⭐️
        # out.collect()가 아니라 yield를 씁니다!
        result_msg = f"State:{state_key}, 10초 평균 온도 :{avg_temp:.2f}°C"
        yield result_msg
    
    # 4. 뒷정리 (초기화)
    self.sum_state.clear()
    self.count_state.clear()
    self.timer_state.clear()
```

---
## Common Beginner Misconceptions (자주 하는 실수)

1. **"Java 예제 보고 `out.collect()` 썼는데 에러나요!"** ❌
    - **정답:** PyFlink의 `ProcessFunction` 계열은 파이썬의 **Generator** 패턴을 사용합니다. 무조건 **`yield`**를 쓰세요.

2. **"함수 인자에 `out`을 넣었는데 인자 개수 안 맞는대요."** ❌
    - **정답:** PyFlink는 `process_element(self, value, ctx)` 딱 3개만 받습니다. `out` 파라미터를 정의부에서 지우세요.

3. **"`return result` 하면 안 되나요?"**
    - **정답:** 안 됩니다. `return`은 함수를 끝내버리지만, 스트림 처리는 계속 데이터를 뱉어야 하므로 `yield`를 써야 합니다.

