---
aliases:
  - Watermark
tags:
  - PyFlink
related:
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_Table_API_Intro]]"
  - "[[PyFlink_SQL_Windows]]"
---
## 한줄요약

"지각생을 얼마나 기다려 줄 것인가?"
네트워크 지연 등으로 순서가 뒤죽박죽인 데이터 스트림에서, "이제 이 시간대 데이터는 다 왔어!"라고 선언하는 기준선)

---
## 왜 필요한가요? (The Problem: Out-of-Order)

현실 세계의 데이터는 **절대 순서대로 도착하지 않습니다.** 
스마트폰이 터널을 지나거나, 서버 네트워크가 잠깐 끊기면 **12:00에 발생한 로그가 12:05에 도착**할 수도 있습니다.

### 비유: 시험 종료 시간

- **상황:** 시험 종료는 **12:00**입니다.
- **Watermark 없음:** 12:00 땡 치자마자 답안지 걷어서 나갑니다. (12:01에 헐레벌떡 뛰어온 학생은 0점)
- **Watermark (5분):** 선생님이 **12:05까지 기다려줍니다.**
    - 12:04에 도착한 학생: "저 12:00에 다 풀었는데 차가 막혀서 늦게 왔어요!" 👉 **인정 (점수 포함)**
    - 12:06에 도착한 학생: 👉 **기각 (너는 너무 늦었다)**

Flink에게 **"데이터가 좀 늦게 와도 5초 정도는 기다렸다가 집계해!"** 라고 알려주는 것이 바로 Watermark입니다.

---
## SQL 문법 (How to Use)

Table API(SQL)에서는 테이블을 만들 때(`CREATE TABLE`) **DDL 구문 안에서 선언**합니다.

```sql
CREATE TABLE user_logs (
    user_id STRING,
    price INT,
    ts TIMESTAMP(3), -- (1) 실제 데이터의 시간 컬럼
    
    -- (2) 워터마크 선언 (핵심!)
    WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    ...
)
```

### 🔍 코드 해부

이때 ts는 내가 가지고 있는 컬럼 만약 컬럼이름이 timestamp이면, `timestamp TIMESTAMP(3)` 이렇게 되는것
**`WATERMARK FOR ts`**: "`ts` 컬럼을 기준으로 시간을 잴 거야." (이때부터 `ts`는 단순한 문자열이 아니라 Flink가 인식하는 **Event Time**이 됩니다.)
**`AS ts - INTERVAL '5' SECOND`**: "내 시계는 실제 데이터 시간보다 **5초 늦게** 가게 할 거야." > 즉, **"최대 5초까지의 지각생(늦게 도착한 데이터)은 봐주겠다"** 는 뜻입니다.

---
## 실전 팁

### Q1. 몇 초로 설정해야 하나요?

- 보통 네트워크 지연을 고려해 **3초 ~ 5초** 정도로 시작합니다.

### Q2. Watermark 선언 안 하면 어떻게 되나요?

- 윈도우 함수(`TUMBLE`, `HOP` 등)를 아예 사용할 수 없습니다.
- 에러 메시지: `Window requires a time attribute`

### Q3. `ts` 컬럼의 타입은?

- 반드시 **`TIMESTAMP(3)`** 타입이어야 합니다.
- Kafka에서 들어오는 데이터가 Unix Time(Long 숫자)이나 String이라면, `TO_TIMESTAMP()` 등을 써서 변환해줘야 합니다.