---
aliases:
  - Evictor
  - Window Evictor
  - 데이터솎아내기
  - 이빅터
  - ProcessWindowFunction
  - 슬라이싱
tags:
  - PyFlink
  - Kafka
related:
  - "[[PyFlink_Trigger_Watermark]]"
  - "[[PyFlink_Windows]]"
  - "[[Python_Generators_Yield]]"
  - "[[Python_Lists_Tuples]]"
  - "[[Python_Iterables_Iterators]]"
linked:
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/kafka_Evictors.py
---
# PyFlink Evictor: 데이터 문지기(가드) / 솎아내기

##  한줄 요약 

**Evictor(이빅터)** 는 윈도우 함수(Reduce, Process 등)가 실행되기 **직전이나 직후**에, 윈도우 안에 있는 데이터 중 **필요 없는 것을 솎아내는(Remove)** 필터링 도구입니다. 

**Java Flink**에는 `CountEvictor`, `TimeEvictor`가 존재합니다.
하지만 **PyFlink**에는 **Evictor 클래스와 `.evictor()` 메서드가 아예 없습니다.**

> PyFlink에서 Evictor 기능(데이터 솎아내기)이 필요하면, **`ProcessWindowFunction`** 내부에서 파이썬의 **리스트 슬라이싱(`list[-N:]`)** 기능을 이용해 직접 구현해야 합니다.
>`ProcessWindowFunction` 에 자세히 알려면 , [[PyFlink_Window_Functions#`ProcessWindowFunction` ?|`ProcessWindowFunction`란?]]  참조 
---
## 왜 필요한가?

 **문제점 (Problem):**
* 윈도우 사이즈가 매우 크면 메모리가 터질 수 있거나 , 전체 데이터가 아닌 **"가장 최근 N개"** 의 데이터만 가지고 계산해야 할 때가 있습니다. 

 **해결책 (Solution):** 
 - 계산(Reduce)을 시작하기 전에 Evictor가 불필요한 데이터를 미리 쫓아내서(Evict), **메모리를 절약하거나 특정 로직(최신 3개만 유지 등)을 구현**합니다. 
 - Java Flink는 `evictor()`를 쓰지만, **PyFlink**는 윈도우가 닫힐 때 모든 데이터를 받아서 **파이썬의 `list[-N:]` 문법**으로 뒤쪽(최신) 데이터만 잘라내어 계산합니다.

---
##  Practical Context

 **"30초 동안 데이터를 모으되, 계산할 때는 가장 늦게 들어온 3개(`CountEvictor(3)`)만 합쳐라"** 는 로직을 구현할 때 씁니다. 

* **실행 순서:** `Window` 생성 → `Trigger` 발사 → **`Evictor` (데이터 솎아내기)** → `Window Function` (계산) → `Result` (자바)
* **실행 순서:** `Window` 생성 → `Trigger` 발사 → **`Process` 함수 진입** → **리스트 뒤에서 3개 자르기 (Manual Evict)** → 합계 계산 → `Result`(PyFlink)

---
##  Code Core Points

- **변경점:** `.evictor(...)`와 `.reduce(...)`를 사용하지 않습니다.
- **핵심 함수:** `.process(KeepLastNFunction(N))`
- **로직:** `process` 함수 안에서 `elements`(전체 데이터)를 리스트로 바꾸고, `[-3:]` 슬라이싱을 통해 솎아냅니다.

---
### `Reduce` vs `Process` 비교 (왜 이걸 써야 해?)

가장 많이 헷갈리는 부분입니다. 표로 정리해 드릴게요.

|**특징**|**ReduceFunction**|**ProcessWindowFunction**|
|---|---|---|
|**방식**|**점진적 (Incremental)**|**일괄 처리 (Batch-like)**|
|**동작**|데이터 1개 올 때마다 `a + b` 계산|윈도우 닫힐 때 **모든 데이터(`elements`)**를 줌|
|**장점**|메모리를 거의 안 씀 (합계 1개만 기억)|**전체 데이터를 보고** 정렬, 자르기 가능|
|**단점**|전체 흐름(순서, 랭킹)을 모름|데이터가 많으면 **메모리 부족(OOM)** 위험|
|**용도**|단순 합계, 최대/최소값|**Evictor 구현**, 랭킹 구하기, 중앙값 계산|


---
##  Detailed Analysis

```python
"""
[PyFlink] Kafka Manual Evictor Example
목표: PyFlink에 없는 Evictor 기능을 ProcessWindowFunction으로 직접 구현하기

입력 예시: "hello-3", "hello-10", "hello-5"
출력 예시: ("hello", 15) -> (최신 3개: 5+10+?? 합계)
"""

import os
from pyflink.common import WatermarkStrategy, SimpleStringSchema
from pyflink.common.typeinfo import Types
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaOffsetsInitializer
from pyflink.datastream.window import TumblingProcessingTimeWindows
from pyflink.datastream.functions import ProcessWindowFunction 
from pyflink.common.time import Time

# ---------------------------------------------------------
# ⭐️ [핵심] Evictor 역할을 대신하는 커스텀 함수
# ---------------------------------------------------------
class KeepLastNFunction(ProcessWindowFunction):
    def __init__(self, n):
        self.n = n  # 몇 개를 남길지 설정 (예: 3)
    
    # 윈도우가 닫힐 때 Flink가 이 함수를 호출합니다.
    def process(self, key, context, elements):
        
        # [Point 1] elements는 'Iterable' 객체입니다.
        # 슬라이싱([-3:])을 하려면 리스트로 변환해야 합니다.
        # (주의: 데이터가 너무 많으면 메모리 부족 이슈 발생 가능)
        data_list = list(elements)
        
        # [Point 2] Evictor 로직 직접 구현 (Slicing)
        # 리스트의 뒤에서부터 N개만 잘라냅니다.
        # 예: [1, 2, 3, 4, 5] -> [-3:] -> [3, 4, 5]
        evicted_list = data_list[-self.n:]
        
        # [Point 3] 계산 로직 (Reduce)
        # item은 ("hello", "3") 튜플 형태이므로, 인덱스 1번을 꺼내서 정수 변환 후 합산
        total_sum=sum([item[1] for item in evicted_list])
        
        # [Point 4] 결과 배출 (Generator)
        # return 대신 yield를 사용해 (Key, 결과)를 내보냅니다.
        # yield (key, total_sum)
		result_msg=f"{key}{self.n}개 합계는 {total_sum}입니다."
		yield result_msg

def run_evictor_example():
    # 1. 환경 설정
    env = StreamExecutionEnvironment.get_execution_environment()
    # (로컬 실행 시 필요한 Kafka Jar 경로 설정)
    jar_path = "file:///opt/flink/lib/flink-sql-connector-kafka-3.1.0-1.18.jar"
    env.add_jars(jar_path)
    
    # 2. Source 설정 (Kafka)
    source = (
        KafkaSource
        .builder()
        .set_bootstrap_servers("kafka:9092")
        .set_topics("input-topic")
        .set_group_id("evictor-group")
        .set_starting_offsets(KafkaOffsetsInitializer.latest())
        .set_value_only_deserializer(SimpleStringSchema())
        .build()
    )
    
    stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")
    
    # 3. Logic 실행
    result_stream = (
        stream
        # 입력 "hello-3"을 분리 -> ("hello", "3")
        .map(lambda x: (x.split("-")[0], x.split("-")[1]), 
             output_type=Types.TUPLE([Types.STRING(), Types.INT()]))
        
        .key_by(lambda x: x[0]) # Key: "hello"
        
        # 30초 윈도우 설정
        .window(TumblingProcessingTimeWindows.of(Time.seconds(30)))
        
        # ⭐️ .evictor() 대신 .process()로 직접 구현한 함수 적용
        .process(KeepLastNFunction(3))
    )
    
    result_stream.print()
    env.execute("Evictor Example")

if __name__ == "__main__":
    run_evictor_example()
```

---
## `.process()`는 **이중 생활**

PyFlink에서 `.process()`는 **이중 생활**을 합니다. 
호출하는 '대상'이 누구냐에 따라 들어가는 함수의 종류가 다릅니다.

### KeyedStream에서 쓸 때 (윈도우 없을 때)

- **대상:** `stream.key_by(...)` 바로 뒤
- **함수:** `KeyedProcessFunction` (타이머, 상태 사용)
- **코드:** `.process(MyKeyedFunction(), output_type=Types.STRING())`

```python
# 윈도우 없이 키별로 실시간 처리
stream.key_by(lambda x: x[0]) \
      .process(AverageProcessFunction(), output_type=Types.STRING()) 
      # 여기서 AverageProcessFunction은 KeyedProcessFunction을 상속받아야 함!
```

### WindowedStream에서 쓸 때 (윈도우 있을 때)

- **대상:** `stream.key_by(...).window(...)` 뒤
- **함수:** `ProcessWindowFunction` (윈도우 데이터 일괄 처리)
- **코드:** `.process(MyWindowFunction(), output_type=Types.STRING())`

```python
# 윈도우 안에서 데이터를 모아 처리
stream.key_by(lambda x: x[0]) \
      .window(TumblingProcessingTimeWindows.of(Time.seconds(30))) \
      .process(KeepLastNFunction(3), output_type=Types.STRING())
      # 여기서 KeepLastNFunction은 ProcessWindowFunction을 상속받아야 함!
```


## 질문사항

**"성능은 괜찮나요?"**

- `ProcessWindowFunction`은 윈도우의 모든 데이터를 메모리에 모았다가 한 번에 처리하므로, 윈도우 사이즈가 너무 크면(수십만 건) 메모리 부족(OOM)이 발생할 수 있습니다. (Java의 Evictor도 동일한 단점을 가짐).
