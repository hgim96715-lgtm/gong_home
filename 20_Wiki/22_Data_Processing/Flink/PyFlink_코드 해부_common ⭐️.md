---
aliases:
  - PyFlink 코드 구조
  - Flink WordCount 분석
  - PyFlink 기초 문법
tags:
  - Flink
related:
  - "[[Flink_Docker_Setup(PyFlink)]]"
  - "[[PyFlink_Import_Analysis]]"
  - "[[Flink_Dashboard_Analysis]]"
  - "[[00_Apache Flink_HomePage]]"
linked:
  - file:///Users/gong/gong_study_de/apache-flink/playground/src/word_count.py
---
## 개념 한 줄 요약 (Concept Summary)

Flink 코드는 **"데이터가 흐르는 수도관(Pipeline)을 조립하는 과정"** 이다.
(물 들어오는 곳(Source) → 정수기 필터(Transform) → 물 나가는 곳(Sink))

---
## 왜 필요한가 (Why)

 **일반 파이썬(Pandas)과의 차이:** 
* 일반 파이썬은 데이터를 **"한 번에 몽땅"** 메모리에 올려서 처리하지만, Flink는 **"끝없이 들어오는 데이터(Stream)"** 를 하나씩 실시간으로 처리하기 위해 특별한 문법을 쓴다.
* **Lazy Evaluation (게으른 실행):** 코드를 짠다고 바로 실행되는 게 아니라, **"설계도만 그려놓고 마지막에 `execute()`라고 외쳐야"** 그때 움직인다. (이걸 이해 못 하면 "왜 실행 안 되지?" 하고 헤맴)

---
##  Practical Context (실무 활용)

* **로그 분석:** 실시간으로 들어오는 에러 로그에서 "Error"라는 단어가 몇 번 나왔는지 카운트.
* **IoT 센서:** 공장 기계 온도 데이터를 실시간으로 읽어서 평균 온도를 계산.

---
##  Code Core Points (5단계 공식)

Flink 프로그램은 무조건 이 순서대로 진행된다. **(암기 필수 ⭐)**

1.  **Environment:** 무대 세팅 (실행 환경 만들기)
2.  **Source:** 입력 (파일, 카프카, 소켓 등에서 읽기)
3.  **Transform:** 가공 (자르고, 맵핑하고, 합치고)
4.  **Sink:** 출력 (파일 저장, DB 저장, 화면 출력)
5.  **Execute:** 발사 (실행 명령)

---
## Detailed Analysis (코드 뜯어보기)

### ① 필수 Import (무조건 박고 시작)

```python
import os
import sys

from pyflink.common import Types, WatermarkStrategy
from pyflink.common.serialization import Encoder
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors import FileSource, StreamFormat, FileSink
```

- `Types`: 파이썬은 변수 타입이 자유롭지만, Flink 엔진(Java)은 깐깐해서 **"이거 문자열이야(STRING)", "이거 숫자야(INT)"**라고 딱 정해줘야 한다. 안 그러면 에러 남.
- `FileSource`, `FileSink`: 파일을 읽고 쓰기 위한 최신 도구.

> 더 많은 import 의미는 [[PyFlink_Import_Analysis |import 탐구]] 확인 


### ② Step 1: 환경 설정 (The Environment)

```python
env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(1)
```

- **`get_execution_environment()`**: "지금 내가 내 노트북에서 돌리나? 서버에서 돌리나?"를 눈치껏 파악해서 판을 깔아주는 함수.
- **`set_parallelism(1)`**: 일꾼(Thread)을 몇 명 쓸지 정함. (1로 하면 순서대로 처리, 여러 개면 병렬 처리)

### ③ Step 2: 소스 (Source - 읽기)

```python
# 포맷이랑 경로를 지정해서 FileSource 생성
source = FileSource.for_record_stream_format(
    StreamFormat.text_line_format(),  # "한 줄씩 읽을게"
    input_path                        # "이 파일에서"
).build()

ds = env.from_source(
    source=source,
    watermark_strategy=WatermarkStrategy.no_watermarks(),
    source_name="file_source"
)
```

- **`StreamFormat.text_line_format()`**: 텍스트 파일을 한 줄씩 읽어오는 표준 포맷.
- **`WatermarkStrategy.no_watermarks()`**: "이 데이터는 실시간이 아니라 이미 저장된 파일이야. 시간 순서 따질 필요 없어." (배치 처리할 때 씀)
- **`ds` (DataStream)**: 이제부터 데이터는 `ds`라는 파이프를 타고 흐름.

#### `{python}FileSource.for_record_stream_format(...)`

> **의미:** "파일 읽는 기계(FileSource)를 조립할 건데, 포맷이랑 위치를 알려줘!"

```python
FileSource.for_record_stream_format(StreamFormat, Path, ...)
```

- `StreamFormat` : 파일 껍데기 말고 **내용물**이 뭔지 알려달라는 것. (`StreamFormat.text_line_format()`: "텍스트 파일이야, 한 줄씩 읽어.")
-  `Path` : 파일이 있는 위치 

#### `{python}env.from_source(...)`

- **의미:** "자, 위에서 만든 소스(기계)를 Flink 환경(env)에 연결할게!"(FileSource를 environment에 등록)
- 전선을 콘센트에 꽂듯 `env`에 등록해야 합니다.

```python
env.from_source(Source, WatermarkStrategy, SourceName)
```

- `Source` : 만든 `source` 변수 (FileSource 객체)
- `WatermarkStrategy` : 스트림 데이터의 시간 기준을 어떻게 잡을지, 파일 읽기(배치)라면 무조건: `WatermarkStrategy.no_watermarks()`
- `SourceName` : Flink 대시보드(웹)에서 보여질 이름.

>**만든다:** `FileSource.어쩌고(포맷, 경로)` 
>**등록한다:** `env.from_source(만든거, 워터마크, 이름)`

### ④ Step 3: 변환 (Transform - 로직) 제너레이터(Generator) 사용!

```python
# [로직] 문장 쪼개기 -> (단어, 1) 만들기 -> 같은 단어 모으기 -> 더하기

# [핵심] 일반 함수가 아니라 'yield'를 쓰는 제너레이터 함수 정의
def split(line):
    for word in line.split():
        yield word  # "하나 찾았다! 자 받아라!" 하고 즉시 던짐

ds = ds.flat_map(split, output_type=Types.STRING()) \
       .map(lambda i: (i, 1), output_type=Types.TUPLE([Types.STRING(), Types.INT()])) \
       .key_by(lambda i: i[0]) \
       .sum(1)
```

- **`key_by` (끼리끼리 묶기)**: SQL의 `GROUP BY`와 똑같음.
- **`sum(1)` (더하기)**:묶인 애들의 1번째 요소(숫자 1)를 계속 더해라.
- `yield`? (왜 return을 안 쓰나?) -> `return list`를 쓰면 문장이 1억 줄일 때 메모리가 터진다. **`yield`** 를 쓰면 단어를 찾는 족족 하나씩 다음 단계로 넘겨주므로 **메모리를 거의 안 쓰고 속도가 빠르다.** (스트림 처리에 최적화됨)

#### `{python}output_type`

- 파이썬 함수가 처리를 끝내고 뱉어내는 결과물이 **문자열인지, 숫자인지** Flink 엔진(Java)에게 미리 신고
- **Python:** "상자 안에 뭐가 들었든 상관없어. 열어봐야 알지." (동적 타입)
- **Java (Flink 엔진):** "상자 크기가 몇 cm인지, 안에 뭐가 들었는지 미리 말 안 하면 처리 안 해!" (정적 타입)
- Flink는 데이터를 네트워크로 주고받을 때 0과 1로 압축(직렬화)한다. -> 이때 **"이게 글자인지 숫자인지"** 알아야 효율적으로 압축할 수 있다. 모르면 에러가 난다.
- `map`, `flat_map` 같은 변환 함수 안에 **옵션(parameter)** 으로 넣어준다.

#### `{python}ds.map(함수_이름, output_type=Types.원하는타입()`

- 튜를은 리스트 `[]` 안에 순서대로 타입을 적어줘야 한다.
- 데이터가 `["a", "b", "c"]` 처럼 리스트로 나갈 때. (단, 리스트 안의 내용물 타입도 정해줘야 함)

```python
# "Hello" 같은 글자를 반환할 때
output_type=Types.STRING()

# 1, 100, -5 같은 정수를 반환할 때 
output_type=Types.INT()

# ("Hello", 1) -> (문자, 숫자)
output_type=Types.TUPLE([Types.STRING(), Types.INT()])

# ["apple", "banana"] -> 문자들이 들어있는 리스트
output_type=Types.LIST(Types.STRING())
```

#### `{python}sum(n)` 

- `sum(n)`은 **"n번째 위치(Index)에 있는 값을 기준으로 합계를 구해라"** 라는 뜻
- Flink에서 데이터가 **튜플(Tuple)** 형태로 흐를 때는 **"이름"이 없고 "순서"만 있기 때문**이다.

```python
# 데이터: ("Apple", 1)
ds.key_by(lambda x: x[0]) # 0번(단어)끼리 묶어라
.sum(1) # 묶인 애들의 1번(개수)을 더해라
```

**② 만약 데이터 구조가 다르다면? (가격 합계)**

```python
# 데이터: ("Item", "Category", 1000) -> (이름, 분류, 가격)
# 인덱스:    0         1         2

ds.key_by(lambda x: x[1])  # 1번(카테고리) 별로 묶어서
  .sum(2)                  # 2번(가격)을 더해라!
```

- 이때 `sum(1)`을 하면 에러가 난다. (1번은 문자열이니까 더할 수 없음)

 **③ sum 말고 다른 것도 되나?**

- `min(1)`: 1번째 값 중 **최솟값**을 남겨라.
- `max(1)`: 1번째 값 중 **최댓값**을 남겨라.
- `min_by(1)` / `max_by(1)`: (고급) 최솟값/최댓값을 가진 **줄 전체**를 남겨라.


### ④ Step 4: 싱크 (Sink) - 파일로 저장하기

```python
# (1) 저장소(Sink) 만들기
output_path = '/opt/flink/playground/data/output'
sink = FileSink.for_row_format(
    base_path=output_path,               # "어디에 저장할까?" (/opt/flink/...)
    encoder=Encoder.simple_string_encoder("UTF-8") # "어떻게 쓸까?" (한글 안 깨지게 UTF-8)
).build()

# (2) 문자열로 변환해서 저장
ds.map(lambda i: f"{i[0]},{i[1]}", output_type=Types.STRING()) \
  .sink_to(sink)
```

- **`Encoder.simple_string_encoder("UTF-8")`**: 데이터를 텍스트 파일로 쓸 때 사용하는 인코더. (직접 `SimpleStringEncoder` 클래스를 쓰지 않고, `Encoder` 도구함에서 꺼내 쓴다.)

>최신 PyFlink에서는 `SimpleStringEncoder`를 직접 꺼내 쓰는 게 막혀 있어서 이걸로 쓰면 에러가 남 (`ImportError`)
>**`Encoder.simple_string_encoder("UTF-8")`** 꼭 !!! 이렇게 해줘야 한다!!!!

- **`sink_to(sink)`**: 예전엔 `ds.print()`로 화면에 뿌렸지만, 실무에선 `sink_to`를 써서 파일이나 DB로 내보낸다.
- **주의할 점**: `sink`에 넣기 전에 데이터를 **"문자열(String)"** 형태로 예쁘게 포장(`map`)해줘야 파일에 깔끔하게 써진다. (튜플 채로 넣으면 깨질 수 있음)


#### `{python}FileSink.for_row_format(...)`

- **의미:** "줄(Row) 단위로 데이터를 쓰는 저장소를 만들겠다." (주로 TXT, CSV 만들 때 씀)

```python
FileSink.for_row_format( 
	base_path, # 1. 어디에? 
	encoder # 2. 어떻게? 
).build() # 3. 조립 끝!
```

- `base_path (필수)` : 파일이 저장될 **폴더 위치**, 문자열(`'/opt/flink/data'`) 또는 `Path` 객체.
- `encoder (필수)` : 데이터를 쓸 때 사용할 **번역기**. `Encoder.simple_string_encoder("UTF-8")` 이걸 안쓰면 한글이 다 깨진다


### ⑤ Step 5: 실행 (Execute)

```python
env.execute('word_count_20260206')
```

- 이 명령어가 실행되는 순간, 위에서 조립한 파이프라인(`Source -> Split -> Sink`)이 Flink 클러스터로 전송되어 작동을 시작한다.

- **역할:** Flink 대시보드(웹)에 표시될 **작업 이름(Job Name)**
- **자유도:** 아무거나 써도 된다.
- **팁**: 서버에는 동시에 수십 개의 job이 돌아가기때문에 이름을 대충 지으면 **"어떤 게 내가 돌린 건지"** 찾을 수가 없다.
	- 보통 **`[프로젝트명] 하는일_날짜`** 형식으로 짓는다. 
	- *예: `env.execute("word_count_20260206")`*

### ⑥ Step 6: 메인 실행부 (Entry Point) - 시동키

```python
if __name__ == '__main__':
    # 여기에 메인 로직 함수를 넣는다.
    word_count()
```

- **설계도 vs 실체:** `def word_count():`로 함수를 정의만 해두면 그건 **"설계도"** 일 뿐, 아무 일도 일어나지 않는다. 
	- 마지막에 `word_count()`라고 **이름을 불러줘야(Call)** 비로소 코드가 실행된다.
- **안전장치:** `if __name__ == '__main__':` 구문은 **"이 파일이 주인공(Main)으로 실행될 때만 작동하라"** 는 뜻이다.
- **왜 쓰는가?**: 이 구문이 없으면, 나중에 다른 파일에서 `import word_count`로 이 파일을 부품처럼 가져다 쓸 때, **의도치 않게 코드가 막 실행되는 사고**가 터질 수 있다. (실무 필수 패턴!)

---
## 결과 해석 

```text
FileSource 객체구나! <pyflink.datastream.connectors.file_system.FileSource object at 0x7ffffe6d39a0>
Job has been submitted with JobID f79b45d7a03fc40e4412d5427d6acb7b
Program execution finished
Job with JobID f79b45d7a03fc40e4412d5427d6acb7b has finished.
Job Runtime: 4553 ms
```

- `print(f"FileSource 객체구나! {source}")` 가 정상 작동했다 
- `Job has been submitted with JobID f79b45d7...` : 설계도가 **Flink 공장(JobManager)** 으로 배달되었다.
	- 주문 접수 완료! 주문 번호(Job ID)는 `f79b...`입니다." (나중에 문제 생기면 이 번호로 조회합니다.)
- `Program execution finished` :  파이썬 스크립트(`word_count.py`)가 할 일을 다 마치고 종료되었습니다.
- `Job with JobID ... has finished.` : Flink 공장에서도 작업을 에러 없이 끝마쳤습니다.
- `Job Runtime: 4553 ms` :  작업에 걸린 시간입니다.

### 폴더 확인해보기

output 폴더가 생성 그 안에 내용이 있다


### UI로 확인해보기 

![[스크린샷 2026-02-06 오후 5.28.10.png]]

![[스크린샷 2026-02-06 오후 5.28.21.png]]


> 더 자세한 내용은 [[Flink_Dashboard_Analysis|Flink 대시보드 해석]] 참고 

---
## 초보자가 하는 실수 

### Q1. `yield` 대신 `return` 써도 되나요?

- `flat_map`에서는 `yield`를 쓰는 것이 **"PyFlink스러운(Pythonic)"** 정석 방법

### Q2. 결과 파일이 왜 여러 개 생기나요? (`part-xxxx`)

- Flink는 **분산 처리** 시스템이기때문 , spark처럼

### Q3. `sink_to`랑 `print`랑 같이 써도 되나요?

- 할수 있지만 `print`는 성능을 떨어뜨릴 수 있으니, 디버깅할 때만 쓰고 실전에서는 빼는 게 좋다.

