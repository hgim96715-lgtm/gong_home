---
aliases:
  - PyFlink Import 분석
  - Flink 필수 모듈
  - StreamFormat 설명
tags:
  - Concept
  - Basics
  - PyFlink
related:
  - "[[PyFlink_코드 해부_common ⭐️]]"
---

#  PyFlink 필수 모듈(Import) 완전 분석

## StreamExecutionEnvironment (환경)

```python
from pyflink.datastream import StreamExecutionEnvironment
```

- Flink 프로그램은 이 객체(`env`)가 없으면 **아무것도 못 합니다.**
- 이 녀석이 있어야 데이터가 흐르는 길(Stream)을 만들고, 병렬 처리(Parallelism)를 설정하고, 마지막에 "실행해!"(`execute`)라고 명령할 수 있습니다.

### Practical (실무 활용)

 `env = StreamExecutionEnvironment.get_execution_environment()`
- 이 한 줄로 내 컴퓨터(로컬)인지 서버(클러스터)인지 알아서 파악하고 판을 깔아줍니다.

---
## Types (타입)

```python
from pyflink.common import Types
```

- **문제:** 파이썬은 변수에 숫자 넣었다가 문자 넣어도 되지만(Dynamic Typing), Flink의 엔진인 자바는 깐깐합니다(Static Typing).
- **해결:** `Types.STRING()`, `Types.INT()` 처럼 명찰을 딱 붙여줘야, 파이썬에서 데이터를 넘길 때 Flink 엔진이 "아, 이거 문자열이구나" 하고 알아먹습니다.
- **안 쓰면?** 직렬화 에러(Serialization Error)가 나서 프로그램이 뻗어버립니다.

### Practical

- `map`이나 `flat_map`을 쓸 때 `output_type=Types.STRING()` 처럼 **결과가 무슨 모양인지** 꼭 알려줘야 합니다.


---
## WatermarkStrategy (워터마크)

```python
from pyflink.common import WatermarkStrategy
```

- **스트림 처리의 핵심:** 실시간 데이터는 순서가 뒤죽박죽일 수 있습니다. (10시에 보낸 데이터가 10시 5분에 도착할 수도 있음)
- 소스(Source)를 만들 때 Flink가 "시간 관리는 어떻게 할까요?"라고 물어보기 때문에 필수로 넣어줘야 합니다.


### ractical

- **파일 처리(Batch):** 이미 저장된 파일은 시간이 중요하지 않으므로 `WatermarkStrategy.no_watermarks()` (워터마크 끔)를 씁니다.
- **실시간 처리(Stream):** `for_monotonous_timestamps()` 등을 써서 시간 순서를 엄격하게 관리합니다.

---
## StreamFormat (포맷)

```python
from pyflink.datastream.connectors import StreamFormat
```

- **"파일 해독기(Decoder)."** 파일의 껍데기가 아니라 **내용물**을 어떻게 읽을지 정하는 도구.
- `FileSource`(파일 읽는 기계)는 파일의 위치만 알지, 그 안이 텍스트인지, CSV인지, 이진 파일(Binary)인지 모릅니다.
- `StreamFormat`이 **"이거 텍스트 파일이야, 한 줄씩 읽어!"** 라고 알려주는 렌즈 역할을 합니다.

### Practical

- **`StreamFormat.text_line_format()`**: 가장 많이 쓰임. 텍스트 파일을 엔터(`\n`) 기준으로 한 줄씩 끊어서 읽어옵니다. ("utf-8" 인코딩이 기본)