---
aliases:
  - PyFlink
  - SQL
  - Window
  - Tumble
  - Session
  - Slide
tags:
  - PyFlink
related:
  - "[[PyFlink_SQL_Integration]]"
  - "[[PyFlink_Table_API_Intro]]"
  - "[[PyFlink_Table_Expressions]]"
  - "[[PyFlink_Trigger_Watermark]]"
---
## 한줄요약 

"흐르는 강물을 양동이(시간)로 퍼담기."
(끝없이 들어오는 데이터 스트림을 1분, 1시간 등 시간 단위로 쪼개서 집계하는 기술)

---
## 왜 Window가 필요한가요?

배치 처리(Spark)는 데이터의 **"끝"** 이 있습니다. 
(어제 데이터 100개 → 합계 끝) 하지만 스트리밍(Flink)은 데이터가 영원히 들어옵니다. 
**"끝"이 없으니 합계를 낼 수가 없죠.** 그래서 우리는 **"가상의 끝"** 을 만들어줍니다.

>"야 Flink야, 평생 데이터를 다 더하지 말고, **1분마다 잘라서** 더해줘."

---
## Import 

```python
# 1. 윈도우 함수 (Tumble, Session, Slide)
from pyflink.table.window import Tumble
```

---
## Window의 3가지 종류 (The Big Three)

Flink SQL에서는 `GROUP BY` 구문 안에 윈도우 함수를 넣어서 사용합니다.

### ① Tumbling Window (텀블링 윈도우)

- **특징:** **겹치지 않음.** 일정한 간격으로 딱딱 끊어짐.
- **비유:** 방 청소를 **매시 정각**마다 하는 것. (12:00~13:00 청소, 13:00~14:00 청소...)
- **용도:** 매분 매출 합계, 시간당 접속자 수. (가장 많이 씀!)

**문법** `{sql}TUMBLE(시간컬럼, 간격)`

```sql
-- 문법: TUMBLE(시간컬럼, 간격)
GROUP BY TUMBLE(ts, INTERVAL '1' MINUTE)
```

### ② Hopping Window (호핑 윈도우 / Sliding)

- **특징:** **겹침.** 윈도우 크기보다 더 자주 결과를 냄.
- **비유:** "최근 1시간 평균"을 **매 5분마다** 알려줘.
- **용도:** 주식 이동평균선(Moving Average), 실시간 급상승 검색어.

**문법** `HOP(시간컬럼, 슬라이드간격, 윈도우크기)`

```sql
-- 문법: HOP(시간컬럼, 슬라이드간격, 윈도우크기)
-- "10분짜리 윈도우를 5분마다 갱신해라"
GROUP BY HOP(ts, INTERVAL '5' MINUTE, INTERVAL '10' MINUTE)
```

### ③ Session Window (세션 윈도우)

- **특징:** **시간이 정해져 있지 않음.** 데이터가 들어오면 유지되고, 끊기면 닫힘.
- **비유:** 웹사이트 로그인 세션. (클릭 안 하고 30분 지나면 로그아웃)
- **용도:** 사용자 체류 시간 분석, 게임 플레이 시간 분석.

**문법**`SESSION(시간컬럼, 유휴시간)`

```sql
-- 문법: SESSION(시간컬럼, 유휴시간)
-- "데이터가 안 들어온 지 30분이 지나면 세션 종료"
GROUP BY SESSION(ts, INTERVAL '30' MINUTE)
```

---
## 핵심 문법 분해(TVF 방식)

Flink 1.13 버전부터 **TVF (Table-Valued Function)** 문법이 표준이 되었습니다. 이 형태를 외워두세요.

```sql
FROM TABLE(
    TUMBLE(
        TABLE source_table,       -- 어떤 테이블?
        DESCRIPTOR(ts),           -- 시간 기준 컬럼은?
        INTERVAL '1' MINUTE       -- 윈도우 크기는?
    )
)
GROUP BY window_start, window_end, category -- TVF 방식은 `window_start`, `window_end`라는 컬럼을 공짜로 만들어준다.
```

>**왜 이렇게 복잡한가요?** 
>예전 방식(`GROUP BY TUMBLE(ts, ...)`)은 윈도우의 시작/끝 시간을 알기 어려웠습니다.
> TVF 방식은 `window_start`, `window_end`라는 컬럼을 **공짜로 만들어주기 때문에** "이게 몇 시부터 몇 시까지의 데이터인지" DB에 저장하기가 훨씬 쉽습니다.

### 쿼리 전체 예시

```sql
SELECT 
    window_start,                      -- Flink 자동 생성 (윈도우 시작 시각)
    window_end,                        -- Flink 자동 생성 (윈도우 종료 시각)
    category, 
    SUM(total_amount) AS total_sales,
    COUNT(order_id)   AS total_orders
FROM TABLE(
    TUMBLE(TABLE source_table, DESCRIPTOR(ts), INTERVAL '1' MINUTE)
)
GROUP BY window_start, window_end, category
```

### `window_start` / `window_end` — 어디서 나왔나?

내가 만드는 컬럼이 아닙니다.
`TUMBLE()` 실행 시 Flink가 **자동으로 생성해서 붙여주는 컬럼**입니다.

```
ts = 12:00:35  →  window_start = 12:00:00,  window_end = 12:01:00
ts = 12:01:10  →  window_start = 12:01:00,  window_end = 12:02:00
```

>`GROUP BY`에 반드시 포함시켜야 "윈도우별" 집계가 됩니다. 빠뜨리면 전체를 하나로 뭉쳐버립니다.

### `DESCRIPTOR(ts)` — 기준 시간 컬럼 지정

"시간 기준으로 윈도우를 쪼갤 컬럼이 `ts`야!" 라고 Flink에게 알려주는 **컬럼 지정자**입니다.

```sql
DESCRIPTOR(ts)           -- CREATE TABLE에서 선언한 컬럼명과 반드시 일치
DESCRIPTOR(event_time)   -- 시간 컬럼 이름이 event_time이라면 이렇게
```

`ts`는 반드시 `CREATE TABLE`에서 `TIMESTAMP(3)` + `WATERMARK`로 선언된 컬럼이어야 합니다. 아니면 에러납니다.

#### INTERVAL 크기 옵션

|표현|의미|
|---|---|
|`INTERVAL '1' MINUTE`|1분 단위|
|`INTERVAL '30' SECOND`|30초 단위|
|`INTERVAL '1' HOUR`|1시간 단위|
|`INTERVAL '1' DAY`|1일 단위|

### Sink 테이블 선언 시 주의사항

```sql
CREATE TABLE print_sink (
    window_start TIMESTAMP(3),   -- TUMBLE이 만든 컬럼 → 타입 맞춰서 선언
    window_end   TIMESTAMP(3),   -- TUMBLE이 만든 컬럼 → 타입 맞춰서 선언
    category     STRING,
    total_sales  DOUBLE,
    total_orders BIGINT          -- COUNT() 결과는 항상 BIGINT
) WITH (
    'connector' = 'print'
)
```

> **⚠️ `COUNT()` 결과는 항상 `BIGINT`로 선언하세요.** 
> Flink의 `COUNT()`는 내부적으로 `BIGINT`를 반환합니다. 
> `INT`로 선언하면 타입 불일치로 INSERT가 실패합니다.

---

## 주의사항 (Pro Tips)

### Watermark 필수

Window 함수는 **시간(Time)** 을 기준으로 동작합니다. 테이블 DDL에 `WATERMARK FOR ts ...` 정의가 없으면 아래 에러가 납니다.
```
Window requires a time attribute
```

### Processing Time vs Event Time

보통 데이터 안에 들어있는 시간(`ts`)을 쓰는 **Event Time**을 사용합니다. Flink가 데이터를 받은 시간을 쓰는 **Processing Time**은 결과가 부정확할 수 있어 잘 안 씁니다.

---
## 전체 흐름 한눈에 보기

```
source_table (Kafka)
        │
        ▼
TUMBLE(TABLE source_table, DESCRIPTOR(ts), INTERVAL '1' MINUTE)
        │  → Flink가 window_start, window_end 자동 생성
        ▼
GROUP BY window_start, window_end, category
        │  → 1분 구간 + 카테고리별로 묶기
        ▼
SUM(total_amount), COUNT(order_id)
        │  → 구간별 집계
        ▼
INSERT INTO print_sink → 결과 출력

```



