---
aliases:
  - 날짜함수
  - NOW
  - DATE_TRUNC
  - EXTRACT
  - TO_CHAR
  - INTERVAL
tags:
  - SQL
related:
  - "[[SQL_Data_Types]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_String_Functions]]"
  - "[[SQL_Numeric_Functions]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Type_Casting]]"
---
# 시간 · 날짜 함수

> **핵심 요약** **"시간을 쪼개고, 더하고, 빼고, 포장하는 데이터 분석의 타임스톤."** 현재 시간을 구하거나, 날짜에서 '월(Month)'만 빼오거나, "오늘부터 3일 뒤"를 계산하는 등 코호트 분석과 시계열 데이터 처리의 핵심입니다.

---

## Why? 왜 필요한가?

**문제점:**

- "2024년 1월 매출만 보고 싶은데, 데이터는 `2024-01-27 14:30:21` 처럼 초 단위까지 있다."
- "사용자의 나이를 계산해야 하는데, 생년월일만 있고 나이 컬럼이 없다."
- "보고서에 `2024-01` 형태로 찍어야 하는데, DB에는 `2024/01/01`로 되어 있다."

**해결책:**

- **`DATE_TRUNC` / `TRUNC`** → 초 단위 시간을 '월' 단위로 잘라서 묶는다.
- **`AGE` / `MONTHS_BETWEEN`** → 두 날짜의 차이를 구한다.
- **`TO_CHAR`** → 날짜 모양을 원하는 형태로 포장한다.

---

## 실무 적용 사례

1. **월별 통계 (Monthly Stats):** 일별 데이터를 월별로 묶어서(`GROUP BY`) 매출 추이 보기.
2. **코호트 분석 (Cohort):** "가입 후 7일 이내에 재구매한 유저" 찾기 (날짜 더하기/빼기).
3. **요일별 분석:** "우리 서비스는 무슨 요일에 가장 접속이 많지?" (`EXTRACT(DOW)`).

---

## 함수 한눈에 비교 (DB별 요약표)

> [!info] DB별 함수 빠른 비교
> 
> |기능|PostgreSQL 🐘|Oracle 🔴|MSSQL 🟩|
> |:--|:--|:--|:--|
> |**현재 시간**|`NOW()`|`SYSDATE`|`GETDATE()`|
> |**날짜 더하기**|`+ INTERVAL '1 month'`|`ADD_MONTHS()`|`DATEADD(month, 1, ...)`|
> |**날짜 차이**|`AGE()` / `-`|`MONTHS_BETWEEN()` / `-`|`DATEDIFF()`|
> |**날짜 추출**|`EXTRACT()`|`EXTRACT()`|`DATEPART()`|
> |**포맷 변환**|`TO_CHAR()`|`TO_CHAR()`|`FORMAT()` / `CONVERT()`|
> |**날짜 자르기**|`DATE_TRUNC()`|`TRUNC()`|`DATETRUNC()`|

---

## ① 현재 시간 구하기

> [!info] DB별 함수 비교
> 
> |DB|함수|반환값|
> |:--|:--|:--|
> |PostgreSQL / MySQL|`NOW()` 또는 `CURRENT_TIMESTAMP`|날짜 + 시간|
> |PostgreSQL / MySQL|`CURRENT_DATE`|날짜만|
> |Oracle|`SYSDATE`|날짜 + 시간|
> |MSSQL|`GETDATE()` / `CURRENT_TIMESTAMP`|날짜 + 시간|

```sql
-- 🐘 PostgreSQL / MySQL
SELECT NOW();               -- 결과: 2026-02-24 10:30:00.123
SELECT CURRENT_DATE;        -- 결과: 2026-02-24

-- 🔴 Oracle
SELECT SYSDATE FROM DUAL;   -- 결과: 2026-02-24 10:30:00

-- 🟩 MSSQL
SELECT GETDATE();           -- 결과: 2026-02-24 10:30:00.123
SELECT CURRENT_TIMESTAMP;   -- GETDATE()와 동일
```

---

## ② 날짜 연산 (더하기 / 빼기)

> [!info] DB별 문법 비교
> 
> |DB|월 더하기|일 더하기|
> |:--|:--|:--|
> |PostgreSQL|`+ INTERVAL '1 month'`|`+ INTERVAL '3 days'` 또는 `+ 3`|
> |Oracle|`ADD_MONTHS(날짜, 1)`|`날짜 + 3`|
> |MSSQL|`DATEADD(month, 1, 날짜)`|`DATEADD(day, 3, 날짜)`|

```sql
-- 🐘 PostgreSQL
SELECT NOW() + INTERVAL '1 month';       -- 1달 뒤
SELECT NOW() - INTERVAL '1 year';        -- 1년 전
SELECT '2026-02-24'::DATE + 3;           -- 3일 뒤

-- 🔴 Oracle
SELECT ADD_MONTHS(SYSDATE, 1) FROM DUAL; -- 1달 뒤
SELECT SYSDATE + 3 FROM DUAL;            -- 3일 뒤
SELECT SYSDATE + (1/24) FROM DUAL;       -- 1시간 뒤 (24시간을 나눠서 더함)

-- 🟩 MSSQL
SELECT DATEADD(month, 1, GETDATE());     -- 1달 뒤
SELECT DATEADD(year, -1, GETDATE());     -- 1년 전 (음수 = 빼기)
SELECT DATEADD(day, 3, GETDATE());       -- 3일 뒤
SELECT DATEADD(hour, 1, GETDATE());      -- 1시간 뒤
```

> [!warning] ⭐ SQLD 빈출 — 말일(End of Month) 계산의 차이
> 
> '1월 31일 + 1달' 처럼 도착하는 달에 해당 일자가 없으면 말일로 처리하는 건 모든 DB가 동일합니다. 하지만 **'2월 28일(말일) + 1달'** 의 결과는 DB마다 다릅니다!
> 
> |DB|2월 28일 + 1달|이유|
> |:--|:--|:--|
> |**Oracle** `ADD_MONTHS`|**3월 31일**|출발일이 말일이면 → 도착일도 무조건 말일|
> |**PostgreSQL / MSSQL**|**3월 28일**|월 숫자만 바꾸고 일은 그대로 유지|
> 
> 💡 Postgres/MSSQL에서 말일을 원하면: MSSQL은 `EOMONTH()`, Postgres는 `DATE_TRUNC + - INTERVAL '1 day'` 사용

> [!tip] 💡 Oracle 날짜 소수 연산 원리
> Oracle에서 날짜에 **정수를 더하면 = 일(Day) 단위**로 계산됩니다.
> 따라서 시간/분/초 단위로 더하려면 **1을 쪼개서** 더해야 합니다.
>
> | 더하는 값 | 계산식 | 의미 |
> |:--|:--|:--|
> | `1` | `1` | 1일 |
>| `1/24` = `1/12/(60/30)` | `1 ÷ 24 = 0.0416...` | 1시간 (하루를 24등분) |
> | `1/24/60` | `1 ÷ 24 ÷ 60 = 0.000694...` | 1분 (1시간을 60등분) |
> | `1/24*(60/10)` | `1 ÷ 24 × 6` | **6시간** (60을 10으로 나눈 몫만큼) |
> | `1/24/60/60` | `1 ÷ 24 ÷ 60 ÷ 60` | 1초 (1분을 60등분) |
> | `1/24/60*10` | `1분 × 10` | **10분** |
>
> ```sql
> SELECT SYSDATE + 1        FROM DUAL; -- 1일 뒤
> SELECT SYSDATE + 1/24     FROM DUAL; -- 1시간 뒤
> SELECT SYSDATE + 1/24/60  FROM DUAL; -- 1분 뒤
> SELECT SYSDATE + 1/24/60/60 FROM DUAL; -- 1초 뒤
>-- 응용: N분 뒤 계산 
>SELECT SYSDATE + 1/24*(60/10) FROM DUAL; -- 6시간 뒤 (60/10 = 6) 
>SELECT SYSDATE + 1/24/60*10 FROM DUAL; -- 10분 뒤 (1분 × 10)
> ```
>
> 💡 **암기 팁:** `하루(1) → 시간(/24) → 분(/60) → 초(/60)` 순서로 나눈다!

**해석법**

|식|해석|결과|
|---|---|---|
|`1/12`|하루를 12등분|**2시간**|
|`60/30`|60분 중 30분의 비율 = **2배**|2|
|`1/12/(60/30)`|2시간을 2로 나눔|**1시간**|

---

## ③ 두 날짜의 간격 구하기

> [!info] DB별 함수 비교
> 
> |DB|함수|반환 타입|
> |:--|:--|:--|
> |PostgreSQL|`AGE(최근, 과거)`|INTERVAL (예: 35 years 9 mons)|
> |PostgreSQL|`DATE - DATE`|정수 (일수)|
> |Oracle|`MONTHS_BETWEEN(최근, 과거)`|실수 (개월 수)|
> |Oracle|`날짜 - 날짜`|실수 (일수)|
> |MSSQL|`DATEDIFF(단위, 과거, 최근)`|정수|

```sql
-- 🐘 PostgreSQL
SELECT AGE('2026-02-24', '1990-05-01');           -- 35 years 9 mons 23 days
SELECT '2026-02-24'::DATE - '2026-02-20'::DATE;   -- 4 (일수 차이)

-- 🔴 Oracle
SELECT MONTHS_BETWEEN(SYSDATE, '1990-05-01') FROM DUAL; -- 개월 수 (실수)
SELECT SYSDATE - TO_DATE('2026-02-20') FROM DUAL;       -- 일수 차이

-- 🟩 MSSQL
SELECT DATEDIFF(day, '1990-05-01', GETDATE());      -- 일수 (정수)
SELECT DATEDIFF(month, '2026-01-01', '2026-02-24'); -- 1 (개월 차이)
SELECT DATEDIFF(hour, '2026-02-24 09:00', GETDATE()); -- 시간 차이
```

> [!warning] MSSQL `DATEDIFF` 순서 주의! `DATEADD`와 다르게 **과거 날짜가 먼저, 최근 날짜가 나중**에 들어갑니다. 순서가 바뀌면 **음수**가 나옵니다.

---

## ④ 날짜 추출 (Extract)

날짜에서 연도, 월, 일 등 **숫자 하나만 쏙 뽑아서 정수(Number)로 반환**합니다.

> [!info] DB별 함수 비교
> 
> |DB|함수|예시|
> |:--|:--|:--|
> |PostgreSQL / Oracle|`EXTRACT(단위 FROM 날짜)`|`EXTRACT(YEAR FROM NOW())`|
> |MSSQL|`DATEPART(단위, 날짜)`|`DATEPART(YEAR, GETDATE())`|
> |MSSQL 단축형|`YEAR()`, `MONTH()`, `DAY()`|`YEAR(GETDATE())`|

```sql
-- 🐘 PostgreSQL / 🔴 Oracle (표준 SQL)
SELECT EXTRACT(YEAR FROM NOW());   -- 2026
SELECT EXTRACT(MONTH FROM NOW());  -- 2
SELECT EXTRACT(DOW FROM NOW());    -- Postgres 전용: 요일 (0:일 ~ 6:토)

-- 🟩 MSSQL
SELECT DATEPART(YEAR, GETDATE());  -- 2026
SELECT DATEPART(dw, GETDATE());    -- 요일 (1:일 ~ 7:토, Postgres와 다름!)
SELECT YEAR(GETDATE());            -- 단축 함수
```

---

## ⑤ 날짜 포맷 변환 (Formatting)

날짜 타입을 **문자열(String)** 로 변환하면서 원하는 형태로 포장합니다.

> [!info] DB별 함수 비교
> 
> |DB|함수|예시|
> |:--|:--|:--|
> |PostgreSQL / Oracle|`TO_CHAR(날짜, '포맷')`|`TO_CHAR(NOW(), 'YYYY-MM-DD')`|
> |MSSQL (최신)|`FORMAT(날짜, '포맷')`|`FORMAT(GETDATE(), 'yyyy-MM-dd')`|
> |MSSQL (고전)|`CONVERT(타입, 날짜, 코드)`|`CONVERT(VARCHAR, GETDATE(), 23)`|

```sql
-- 🐘 PostgreSQL / 🔴 Oracle
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD');   -- '2026-01-27'
SELECT TO_CHAR(NOW(), 'YYYY년 MM월'); -- '2026년 01월'
SELECT TO_CHAR(NOW(), 'Day');          -- 요일 영문 이름 (예: Tuesday)

-- 🟩 MSSQL
-- FORMAT (직관적, 최신 방식)
SELECT FORMAT(GETDATE(), 'yyyy-MM-dd');  -- 2026-02-24
SELECT FORMAT(GETDATE(), 'yyyy년 MM월'); -- 2026년 02월

-- CONVERT (실무 고전 방식, 코드 암기 필요)
SELECT CONVERT(VARCHAR, GETDATE(), 23);  -- 2026-02-24      ← 가장 많이 씀!
SELECT CONVERT(VARCHAR, GETDATE(), 112); -- 20260224        ← 파일명/로그용
SELECT CONVERT(VARCHAR, GETDATE(), 120); -- 2026-02-24 10:30:00
```

---

## ⑥ EXTRACT vs DATE_TRUNC — 가장 많이 헷갈리는 차이

> [!tip] 핵심 비교
> 
> |함수|결과 타입|용도|
> |:--|:--|:--|
> |**`EXTRACT`**|**숫자 (Number)**|날짜에서 부품 하나만 뽑아냄|
> |**`DATE_TRUNC`** / **`TRUNC`**|**날짜 (Date)**|날짜 형태 유지하되, 지정 단위 이하를 01일 00시로 초기화|

```sql
-- EXTRACT: 숫자 하나만 뽑기
SELECT EXTRACT(MONTH FROM '2026-03-15');   -- 결과: 3 (숫자)

-- DATE_TRUNC: 날짜 형태 유지하며 자르기 (PostgreSQL)
SELECT DATE_TRUNC('month', '2026-03-15'); -- 결과: 2026-03-01 00:00:00 (날짜)

-- TRUNC: Oracle 버전
SELECT TRUNC(SYSDATE, 'MM') FROM DUAL;    -- 결과: 2026-03-01
```

> [!warning] ❌ GROUP BY에서 흔한 대참사
> 
> ```sql
> -- ❌ 이렇게 하면 2023년 1월 + 2024년 1월이 같은 '1'로 합쳐짐!
> GROUP BY EXTRACT(MONTH FROM date)
> 
> -- ✅ 연/월을 안전하게 묶으려면 DATE_TRUNC 사용
> GROUP BY DATE_TRUNC('month', date)        -- PostgreSQL
> GROUP BY TRUNC(date, 'MM')               -- Oracle
> ```

---

## 초보자가 자주 하는 실수

### ① 문자열에 숫자를 더했어요

```sql
-- ❌ 에러!
SELECT '2024-01-01' + 1;

-- ✅ 날짜로 캐스팅 후 더하기
SELECT '2024-01-01'::DATE + 1;  -- PostgreSQL
SELECT TO_DATE('2024-01-01', 'YYYY-MM-DD') + 1 FROM DUAL;  -- Oracle
```

> [!note] 📎 형변환(Type Casting) 더 알아보기
> 날짜 연산에서 타입이 맞지 않으면 에러가 발생합니다.
>
> | 구분 | 설명 | 예시 |
> |:--|:--|:--|
> | **명시적 형변환** | 개발자가 직접 타입을 지정 | `'2024-01-01'::DATE` / `TO_DATE(...)` |
> | **암시적 형변환** | DB가 자동으로 타입을 변환 | `'2024-01-01' + 1` → DB가 알아서 날짜로 변환 시도 |
>
> ⚠️ 암시적 형변환은 DB마다 동작이 달라 **예상치 못한 결과**가 나올 수 있습니다.
> 날짜 연산에서는 항상 **명시적 형변환**을 권장합니다.
>
> 👉 자세한 내용은 [[SQL_Type_Casting]] 참조

### ② 타임존 때문에 날짜가 달라져요

서버 시간은 UTC(영국 기준)인데 한국(KST, +9)으로 생각하고 쿼리를 짜면, 오전 9시 이전 데이터는 **'어제'** 로 잡힙니다.

```sql
-- ✅ 타임존 명시
SELECT NOW() AT TIME ZONE 'Asia/Seoul';
```

### ③ 두 날짜를 뺐는데 이상한 값이 나와요

|연산|결과 타입|예시|
|:--|:--|:--|
|`TIMESTAMP - TIMESTAMP`|INTERVAL|`3 days 04:00:00`|
|`DATE - DATE`|INTEGER|`3`|

D-Day 계산할 거라면 둘 다 `::DATE`로 맞춰서 빼는 게 안전합니다.

### ④ TO_DATE에 연월만 넣었더니 범위 조회가 안 돼요 ⭐ SQLD 빈출

> [!warning] TO_DATE 함정
> `TO_DATE('201501', 'YYYYMM')` 을 변환하면
> → **2015-01-01 00:00:00** 딱 그 한 순간의 값이 됩니다.
>
> | 조건 | 실제 조회 범위 |
> |:--|:--|
> | `TO_CHAR(날짜, 'YYYYMM') = '201501'` | ✅ 2015년 1월 **전체** |
> | `날짜 >= TO_DATE('20150101') AND 날짜 < TO_DATE('20150201')` | ✅ 2015년 1월 **전체** |
> | `TO_DATE('201501', 'YYYYMM') = 날짜` | ❌ 2015-01-01 00:00:00 **딱 그 순간만** |
>
> 월 전체를 조회하려면 반드시 **범위 조건(`>=`, `<`)** 이나 **`TO_CHAR` 변환**을 사용하세요!




