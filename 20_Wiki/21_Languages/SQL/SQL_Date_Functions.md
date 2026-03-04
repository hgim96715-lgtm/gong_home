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
# SQL Date Functions — 시간 · 날짜 함수

## 개념 한 줄 요약

> **"시간을 쪼개고, 더하고, 빼고, 포장하는 데이터 분석의 타임스톤."** 현재 시간 조회, 날짜에서 월만 추출, "오늘부터 3일 뒤" 계산 등 코호트 분석과 시계열 처리의 핵심이다.

---

## 왜 필요한가?

|문제|해결 함수|
|---|---|
|`2024-01-27 14:30:21` 에서 월 단위로만 묶고 싶다|`DATE_TRUNC` / `TRUNC`|
|생년월일로 나이를 계산해야 한다|`AGE` / `MONTHS_BETWEEN`|
|DB 날짜를 `2024-01` 형태로 출력해야 한다|`TO_CHAR`|

**실무 적용 사례:**

- 월별 매출 추이: 일별 데이터를 월별로 묶어 `GROUP BY`
- 코호트 분석: "가입 후 7일 이내 재구매한 유저" 찾기
- 요일별 분석: "무슨 요일에 접속이 가장 많지?" (`EXTRACT(DOW)`)

---

## DB별 함수 한눈에 비교

|기능|PostgreSQL 🐘|Oracle 🔴|MSSQL 🟩|
|---|---|---|---|
|현재 시간|`NOW()`|`SYSDATE`|`GETDATE()`|
|날짜 더하기|`+ INTERVAL '1 month'`|`ADD_MONTHS()`|`DATEADD(month, 1, ...)`|
|날짜 차이|`AGE()` / `-`|`MONTHS_BETWEEN()` / `-`|`DATEDIFF()`|
|날짜 추출|`EXTRACT()`|`EXTRACT()`|`DATEPART()`|
|포맷 변환|`TO_CHAR()`|`TO_CHAR()`|`FORMAT()` / `CONVERT()`|
|날짜 자르기|`DATE_TRUNC()`|`TRUNC()`|`DATETRUNC()`|

---

---

# ① 현재 시간 구하기

|DB|함수|반환값|
|---|---|---|
|PostgreSQL / MySQL|`NOW()` 또는 `CURRENT_TIMESTAMP`|날짜 + 시간|
|PostgreSQL / MySQL|`CURRENT_DATE`|날짜만|
|Oracle|`SYSDATE`|날짜 + 시간|
|MSSQL|`GETDATE()` / `CURRENT_TIMESTAMP`|날짜 + 시간|

```sql
-- 🐘 PostgreSQL / MySQL
SELECT NOW();               -- 2026-02-24 10:30:00.123
SELECT CURRENT_DATE;        -- 2026-02-24

-- 🔴 Oracle
SELECT SYSDATE FROM DUAL;   -- 2026-02-24 10:30:00

-- 🟩 MSSQL
SELECT GETDATE();           -- 2026-02-24 10:30:00.123
SELECT CURRENT_TIMESTAMP;   -- GETDATE()와 동일
```

---

---

# ② 날짜 연산 (더하기 / 빼기)

|DB|월 더하기|일 더하기|
|---|---|---|
|PostgreSQL|`+ INTERVAL '1 month'`|`+ INTERVAL '3 days'` 또는 `+ 3`|
|Oracle|`ADD_MONTHS(날짜, 1)`|`날짜 + 3`|
|MSSQL|`DATEADD(month, 1, 날짜)`|`DATEADD(day, 3, 날짜)`|

```sql
-- 🐘 PostgreSQL
SELECT NOW() + INTERVAL '1 month';        -- 1달 뒤
SELECT NOW() - INTERVAL '1 year';         -- 1년 전
SELECT '2026-02-24'::DATE + 3;            -- 3일 뒤

-- 🔴 Oracle
SELECT ADD_MONTHS(SYSDATE, 1) FROM DUAL;  -- 1달 뒤
SELECT SYSDATE + 3 FROM DUAL;             -- 3일 뒤
SELECT SYSDATE + (1/24) FROM DUAL;        -- 1시간 뒤

-- 🟩 MSSQL
SELECT DATEADD(month, 1, GETDATE());      -- 1달 뒤
SELECT DATEADD(year, -1, GETDATE());      -- 1년 전 (음수 = 빼기)
SELECT DATEADD(day, 3, GETDATE());        -- 3일 뒤
SELECT DATEADD(hour, 1, GETDATE());       -- 1시간 뒤
```

---

##  INTERVAL 연산 후 시간(00:00) 제거하기 ⭐️

`DATE + INTERVAL` 을 하면 결과가 **TIMESTAMP** 로 반환되어 뒤에 `00:00:00` 이 붙는다.

```sql
-- ❌ INTERVAL 연산 결과 → TIMESTAMP 로 반환됨
SELECT '20211231'::DATE - INTERVAL '3 month';
-- 결과: 2021-09-30 00:00:00  ← 뒤에 00:00:00 이 붙음!
```

**이유:** `DATE - INTERVAL` 연산에서 DB 가 "혹시 시간 정보가 있을 수도 있으니까" 안전하게 TIMESTAMP 타입으로 업캐스팅해서 반환하기 때문이다.

**해결법: 결과를 다시 `::DATE` 로 캐스팅**

```sql
-- ✅ 뒤에 ::DATE 를 한 번 더 붙여서 날짜만 남긴다
SELECT ('20211231'::DATE - INTERVAL '3 month')::DATE;
-- 결과: 2021-09-30  ← 깔끔!

-- 괄호가 핵심: INTERVAL 연산 전체를 감싸고 나서 ::DATE 를 붙인다
-- ❌ 괄호 없이 쓰면 의도와 다르게 동작할 수 있음
SELECT '20211231'::DATE - INTERVAL '3 month'::DATE;  -- 에러 또는 오작동
```

**실무에서 자주 쓰는 패턴:**

```sql
-- 기준일 기준 3개월 이내 입사자 필터링 (멘토링 문제 등)
WHERE hire_date > ('20211231'::DATE - INTERVAL '3 month')::DATE
-- 결과: hire_date > 2021-09-30

-- 기준일 기준 재직 2년 이상인 직원 필터링
WHERE hire_date <= ('20211231'::DATE - INTERVAL '2 year')::DATE
-- 결과: hire_date <= 2019-12-31

-- 최근 30일 데이터 조회
WHERE created_at >= (CURRENT_DATE - INTERVAL '30 days')::DATE

-- 1년 전 날짜 변수처럼 쓰기
SELECT (NOW() - INTERVAL '1 year')::DATE AS one_year_ago;
-- 결과: 2025-02-24  (시간 없이 깔끔)
```

>**💡 재직 기간 조건 해석법** "기준일 기준 N년 이상 재직" 조건은 항상 아래 공식으로 바꿔 생각한다.

```text
재직 기간 조건   →  입사일 ≤ 기준일 − 기간

"2년 이상 재직"   →  hire_date ≤ 2021-12-31 − 2년   →  hire_date ≤ 2019-12-31
"3개월 이내 입사" →  hire_date > 2021-12-31 − 3개월  →  hire_date > 2021-09-30
```

>왜 `<=` 인가: 입사일이 **오래될수록(작을수록)** 재직 기간이 길다. "2년 이상"이 되려면 입사일이 기준일에서 2년을 뺀 날짜보다 **같거나 더 과거** 여야 한다.




> **💡 핵심 암기법** `DATE ± INTERVAL` → 결과가 TIMESTAMP 로 튀어나온다 → 뒤에 `::DATE` 로 다시 잡아준다

 ```
 ('기준날짜'::DATE  ±  INTERVAL 'N unit')::DATE
  ─────────────────────────────────────────────
         연산 전체를 괄호로 감싸고
         맨 끝에 ::DATE 를 한 번 더!
 ```

---

## ⭐ SQLD 빈출 — 말일(End of Month) 계산의 차이

'1월 31일 + 1달' 처럼 도착 달에 해당 일자가 없으면 말일로 처리하는 건 모든 DB 가 동일하다. 하지만 **'2월 28일(말일) + 1달'** 의 결과는 DB 마다 다르다.

|DB|2월 28일 + 1달|이유|
|---|---|---|
|Oracle `ADD_MONTHS`|**3월 31일**|출발일이 말일이면 → 도착일도 무조건 말일|
|PostgreSQL / MSSQL|**3월 28일**|월 숫자만 바꾸고 일은 그대로 유지|

Postgres/MSSQL 에서 말일을 원하면: MSSQL 은 `EOMONTH()`, Postgres 는 `DATE_TRUNC + - INTERVAL '1 day'` 사용.

---

## Oracle 날짜 소수 연산 원리

Oracle 에서 날짜에 **정수를 더하면 = 일(Day) 단위** 로 계산된다. 시간/분/초 단위로 더하려면 **1을 쪼개서** 더해야 한다.

|더하는 값|계산식|의미|
|---|---|---|
|`1`|`1`|1일|
|`1/24`|`1 ÷ 24`|1시간|
|`1/24/60`|`1 ÷ 24 ÷ 60`|1분|
|`1/24/60/60`|`1 ÷ 24 ÷ 60 ÷ 60`|1초|

```sql
SELECT SYSDATE + 1         FROM DUAL;  -- 1일 뒤
SELECT SYSDATE + 1/24      FROM DUAL;  -- 1시간 뒤
SELECT SYSDATE + 1/24/60   FROM DUAL;  -- 1분 뒤
SELECT SYSDATE + 1/24/60/60 FROM DUAL; -- 1초 뒤

-- 응용
SELECT SYSDATE + 1/24*6    FROM DUAL;  -- 6시간 뒤
SELECT SYSDATE + 1/24/60*10 FROM DUAL; -- 10분 뒤
```

> **암기 팁:** `하루(1) → 시간(/24) → 분(/60) → 초(/60)` 순서로 나눈다.

---

---

# ③ 두 날짜의 간격 구하기

|DB|함수|반환 타입|
|---|---|---|
|PostgreSQL|`AGE(최근, 과거)`|INTERVAL (예: 35 years 9 mons)|
|PostgreSQL|`DATE - DATE`|정수 (일수)|
|Oracle|`MONTHS_BETWEEN(최근, 과거)`|실수 (개월 수)|
|Oracle|`날짜 - 날짜`|실수 (일수)|
|MSSQL|`DATEDIFF(단위, 과거, 최근)`|정수|

```sql
-- 🐘 PostgreSQL
SELECT AGE('2026-02-24', '1990-05-01');
-- 35 years 9 mons 23 days

SELECT '2026-02-24'::DATE - '2026-02-20'::DATE;
-- 4 (일수 차이)

-- 🔴 Oracle
SELECT MONTHS_BETWEEN(SYSDATE, '1990-05-01') FROM DUAL;
-- 개월 수 (실수)

SELECT SYSDATE - TO_DATE('2026-02-20') FROM DUAL;
-- 일수 차이

-- 🟩 MSSQL
SELECT DATEDIFF(day, '1990-05-01', GETDATE());       -- 일수
SELECT DATEDIFF(month, '2026-01-01', '2026-02-24');  -- 1 (개월 차이)
SELECT DATEDIFF(hour, '2026-02-24 09:00', GETDATE()); -- 시간 차이
```

> ⚠️ **MSSQL `DATEDIFF` 순서 주의!** `DATEADD` 와 다르게 **과거 날짜가 먼저, 최근 날짜가 나중** 에 들어간다. 순서가 바뀌면 **음수** 가 나온다.

---

---

# ④ 날짜 추출 (EXTRACT)

날짜에서 연도, 월, 일 등 **숫자 하나만 뽑아서 정수(Number)로 반환** 한다.

|DB|함수|예시|
|---|---|---|
|PostgreSQL / Oracle|`EXTRACT(단위 FROM 날짜)`|`EXTRACT(YEAR FROM NOW())`|
|MSSQL|`DATEPART(단위, 날짜)`|`DATEPART(YEAR, GETDATE())`|
|MSSQL 단축형|`YEAR()`, `MONTH()`, `DAY()`|`YEAR(GETDATE())`|

```sql
-- 🐘 PostgreSQL / 🔴 Oracle
SELECT EXTRACT(YEAR FROM NOW());   -- 2026
SELECT EXTRACT(MONTH FROM NOW());  -- 2
SELECT EXTRACT(DOW FROM NOW());    -- 요일 (0:일 ~ 6:토, PostgreSQL 전용)

-- 🟩 MSSQL
SELECT DATEPART(YEAR, GETDATE());  -- 2026
SELECT DATEPART(dw, GETDATE());    -- 요일 (1:일 ~ 7:토 ← Postgres 와 숫자 다름!)
SELECT YEAR(GETDATE());            -- 단축 함수
```

---

---

# ⑤ 날짜 포맷 변환 (TO_CHAR)

날짜 타입을 **문자열(String)** 로 변환하면서 원하는 형태로 포장한다.

|DB|함수|예시|
|---|---|---|
|PostgreSQL / Oracle|`TO_CHAR(날짜, '포맷')`|`TO_CHAR(NOW(), 'YYYY-MM-DD')`|
|MSSQL (최신)|`FORMAT(날짜, '포맷')`|`FORMAT(GETDATE(), 'yyyy-MM-dd')`|
|MSSQL (고전)|`CONVERT(타입, 날짜, 코드)`|`CONVERT(VARCHAR, GETDATE(), 23)`|

```sql
-- 🐘 PostgreSQL / 🔴 Oracle
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD');    -- '2026-01-27'
SELECT TO_CHAR(NOW(), 'YYYY년 MM월');  -- '2026년 01월'
SELECT TO_CHAR(NOW(), 'Day');           -- 요일 영문 이름 (예: Tuesday)

-- 🟩 MSSQL
SELECT FORMAT(GETDATE(), 'yyyy-MM-dd');   -- 2026-02-24
SELECT FORMAT(GETDATE(), 'yyyy년 MM월'); -- 2026년 02월

-- CONVERT (실무 고전 방식, 코드 암기 필요)
SELECT CONVERT(VARCHAR, GETDATE(), 23);   -- 2026-02-24      ← 가장 많이 씀
SELECT CONVERT(VARCHAR, GETDATE(), 112);  -- 20260224        ← 파일명·로그용
SELECT CONVERT(VARCHAR, GETDATE(), 120);  -- 2026-02-24 10:30:00
```

---

---

# ⑥ EXTRACT vs DATE_TRUNC — 가장 많이 헷갈리는 차이

|함수|결과 타입|용도|
|---|---|---|
|`EXTRACT`|**숫자 (Number)**|날짜에서 부품 하나만 뽑아냄|
|`DATE_TRUNC` / `TRUNC`|**날짜 (Date)**|날짜 형태 유지, 지정 단위 이하를 01일 00시로 초기화|

```sql
-- EXTRACT: 숫자 하나만 뽑기
SELECT EXTRACT(MONTH FROM '2026-03-15');    -- 결과: 3 (숫자)

-- DATE_TRUNC: 날짜 형태 유지하며 자르기 (PostgreSQL)
SELECT DATE_TRUNC('month', '2026-03-15');  -- 결과: 2026-03-01 00:00:00

-- TRUNC: Oracle 버전
SELECT TRUNC(SYSDATE, 'MM') FROM DUAL;    -- 결과: 2026-03-01
```

### ⚠️ GROUP BY 에서 흔한 실수

```sql
-- ❌ 2023년 1월 + 2024년 1월이 같은 '1'로 합쳐짐!
GROUP BY EXTRACT(MONTH FROM date)

-- ✅ 연·월을 안전하게 묶으려면 DATE_TRUNC 사용
GROUP BY DATE_TRUNC('month', date)   -- PostgreSQL
GROUP BY TRUNC(date, 'MM')          -- Oracle
```

---

---

# 초보자가 자주 하는 실수

## ① 문자열에 숫자를 더했어요

```sql
-- ❌ 에러!
SELECT '2024-01-01' + 1;

-- ✅ 날짜로 캐스팅 후 더하기
SELECT '2024-01-01'::DATE + 1;                              -- PostgreSQL
SELECT TO_DATE('2024-01-01', 'YYYY-MM-DD') + 1 FROM DUAL;  -- Oracle
```

> 형변환 자세한 내용 → [[SQL_Type_Casting]] 참조

---

## ② INTERVAL 연산 후 00:00:00 이 붙어요 ⭐️

위 섹션 [[SQL_Date_Functions#INTERVAL 연산 후 시간(00 00) 제거하기 ⭐️|NTERVAL 연산 후 시간(00 00) 제거하기]] 참고

```sql
-- 핵심만 기억: 결과 전체를 괄호로 감싸고 ::DATE 를 붙인다
SELECT ('20211231'::DATE - INTERVAL '3 month')::DATE;
-- 2021-09-30  ✅
```

---

## ③ 타임존 때문에 날짜가 달라져요

서버 시간이 UTC 인데 한국(KST, +9) 기준으로 쿼리를 짜면, 오전 9시 이전 데이터는 **'어제'** 로 잡힌다.

```sql
-- ✅ 타임존 명시
SELECT NOW() AT TIME ZONE 'Asia/Seoul';
```

---

## ④ 두 날짜를 뺐는데 이상한 값이 나와요

|연산|결과 타입|예시|
|---|---|---|
|`TIMESTAMP - TIMESTAMP`|INTERVAL|`3 days 04:00:00`|
|`DATE - DATE`|INTEGER|`3`|

D-Day 계산할 거라면 둘 다 `::DATE` 로 맞춰서 빼는 게 안전하다.

---

## ⑤ TO_DATE 에 연월만 넣었더니 범위 조회가 안 돼요 ⭐️ SQLD 빈출

`TO_DATE('201501', 'YYYYMM')` 을 변환하면 **2015-01-01 00:00:00** 딱 그 한 순간의 값이 된다.

|조건|실제 조회 범위|
|---|---|
|`TO_CHAR(날짜, 'YYYYMM') = '201501'`|✅ 2015년 1월 **전체**|
|`날짜 >= TO_DATE('20150101') AND 날짜 < TO_DATE('20150201')`|✅ 2015년 1월 **전체**|
|`TO_DATE('201501', 'YYYYMM') = 날짜`|❌ 2015-01-01 00:00:00 **딱 그 순간만**|

월 전체를 조회하려면 반드시 **범위 조건(`>=`, `<`)** 이나 **`TO_CHAR` 변환** 을 사용하자.

---

