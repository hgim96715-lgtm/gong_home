---
aliases:
  - Spark Streaming
  - Structured Streaming
  - DStream
tags:
  - Spark
  - Streaming
related:
  - "[[Spark_DataFrame_SQL_Intro]]"
  - "[[Spark_Architecture]]"
  - "[[00_Apache_Spark_HomePage]]"
---
##  개념 한 줄 요약 

**"끊임없이 들어오는 데이터(Stream)를 아주 짧게 잘라서(Micro-batch) 배치처럼 처리하는 기술."**

* 기존의 스파크가 "고인 물(저장된 파일)"을 퍼내는 펌프라면,
* 스파크 스트리밍은 **"흐르는 수돗물(실시간 로그)"** 을 받아내는 파이프라인입니다
* **Kafka, Flume** 같은 곳에서 데이터를 받아 DB나 대시보드로 쏴줍니다

---
##  두 가지 처리 방식 (Legacy vs Modern) 

스파크 스트리밍에는 족보가 두 개 있습니다. 무조건 **신세대(Structured)** 를 써야 하지만, 역사는 알아야 합니다.

### ①  Spark Streaming (DStream) - "구세대"

* **기반:** **RDD** (저수준 API)[
* **작동 원리:** 들어오는 데이터를 시간 단위(예: 1초)로 툭툭 잘라서 **작은 RDD 뭉치(Micro-batch)** 로 만듭니다
* **단점:**
    * 코딩이 어렵습니다 (RDD 메서드 사용)
    * 스키마(Schema)를 몰라서 최적화가 안 됩니다.
    * 이벤트 시간(Event Time) 처리가 복잡합니다

### ② Structured Streaming - "신세대" 

* **기반:** **DataFrame / SQL** (고수준 API)
* **작동 원리:** 스트림 데이터를 **"끝없이 늘어나는 무한한 테이블(Unbounded Table)"** 로 취급합니다
* **장점:**
    * **쉽다:** 배치 처리할 때 썼던 DataFrame 코드를 그대로 씁니다
    * **똑똑하다:** Catalyst Optimizer가 최적화를 대신 해줍니다
    * **정확하다:** 늦게 도착한 데이터(Late Data)도 **Event Time** 기준으로 똑똑하게 처리합니다

---
## 무한한 테이블 (Unbounded Table)

Structured Streaming의 핵심 아이디어입니다.

* **Input:** 새로운 데이터가 들어온다 = **"테이블에 새로운 행(Row)이 추가(Append)된다"** 
* **Query:** 사용자는 그냥 "이 테이블에 SQL 날려!" 라고 명령합니다.
* **Output:** 스파크가 알아서 새로운 데이터만 계산해서 결과를 갱신합니다.

> **💡 비유:**
> * **DStream:** 1초마다 작은 물컵에 물을 받아서 마시는 것.
> * **Structured Streaming:** 수도꼭지를 틀어놓고, 흐르는 물 전체를 하나의 긴 강물(Table)로 보고 분석하는 것.

---
## 윈도우(Window)와 지연 데이터(Late Data) 

실시간 처리는 "데이터가 생성된 시간"과 "도착한 시간"이 다를 수 있습니다.

* **Event Time:** 데이터가 실제 발생한 시간 (예: 스마트폰에서 클릭한 시간)
* **Processing Time:** 데이터가 서버에 도착해서 처리된 시간.
* **Watermarking:** "너무 늦게 온 데이터는 버린다"는 기준선.
    * 예: "지금 12:10분인데, 12:00분 데이터가 이제 왔어? 10분 지났으니 무시해!" (Late Data Handling)

---
### Tip

>"면접관이 '스파크 스트리밍 써보셨나요?' 하면 꼭 되물어봐야 해.
>'**DStream 방식인가요, Structured Streaming 방식인가요?**' 라고.
>현업에서는 이제 99% **Structured Streaming**을 쓰니까, RDD 기반의 DStream은 '아, 옛날엔 그랬지' 정도만 알아두면 돼!"