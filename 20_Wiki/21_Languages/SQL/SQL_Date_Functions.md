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
  - "[[Window_Functions]]"
  - "[[SQL_String_Functions]]"
  - "[[SQL_Numeric_Functions]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Type_Casting]]"
---
# 시간 , 날짜 함수

## 개념 한 줄 요약

**"시간을 쪼개고, 더하고, 빼고, 포장하는 데이터 분석의 타임스톤."**
현재 시간을 구하거나, 날짜에서 '월(Month)'만 빼오거나, "오늘부터 3일 뒤"를 계산하는 등 코호트 분석과 시계열 데이터 처리의 핵심입니다.

---
## 왜 필요한가? (Why)

**문제점:**
- "2024년 1월 매출만 보고 싶은데, 데이터는 '2024-01-27 14:30:21' 처럼 초 단위까지 있다."
- "사용자의 나이를 계산해야 하는데, 생년월일만 있고 나이 컬럼이 없다."
- "보고서에 '2024-01' 형태로 찍어야 하는데, DB에는 '2024/01/01'로 되어 있다."

**해결책:**
- **`DATE_TRUNC` / `TRUNC`** 로 초 단위 시간을 '월' 단위로 잘라내서 묶는다.
- **`AGE` / `MONTHS_BETWEEN`** 으로 두 날짜의 차이를 구한다.
- **`TO_CHAR`** 로 날짜 모양을 예쁘게 포장한다.

---

## 실무 적용 사례 (Practical Context)

1.  **월별 통계 (Monthly Stats):** 일별 데이터를 월별로 묶어서(`GROUP BY`) 매출 추이 보기.
2.  **코호트 분석 (Cohort):** "가입 후 7일 이내에 재구매한 유저" 찾기 (날짜 더하기/빼기).
3.  **요일별 분석:** "우리 서비스는 무슨 요일에 가장 접속이 많지?" (`EXTRACT(DOW)`).

---
##  Code Core Points: 필수 함수 4대장 (PostgreSQL vs Oracle 완벽 비교)

### ① 현재 시간 구하기 (Current Date & Time)

데이터베이스마다 현재 시간을 호출하는 함수가 다릅니다. 반환되는 결과에 '시간'이 포함되는지, '날짜'만 나오는지가 핵심입니다.

**PostgreSQL / MySQL**
- `NOW()` 또는 `CURRENT_TIMESTAMP`: 현재 **날짜와 시간**을 모두 반환합니다. (Type: Timestamp)
- `CURRENT_DATE`: 시간은 버리고 **오늘 날짜**만 반환합니다. (Type: Date)

**Oracle (SQLD 필수)**
- `SYSDATE`: 현재 **날짜와 시간**을 모두 반환합니다. (Type: Date)

**SQL Server (MSSQL)**
-  `GETDATE()`: 현재 **날짜와 시간**을 모두 반환합니다. Oracle의 `SYSDATE`, Postgres의 `NOW()`와 완벽히 같은 역할입니다! 
- `CURRENT_TIMESTAMP`: ANSI 표준 함수로, `GETDATE()`와 완벽히 동일한 결과를 반환합니다.

```sql
-- 🐘 PostgreSQL / MySQL
SELECT NOW();               -- 결과: 2026-02-24 10:30:00.123
SELECT CURRENT_DATE;        -- 결과: 2026-02-24

-- 🔴 Oracle (SQLD 단골 출제)
SELECT SYSDATE FROM DUAL;   -- 결과: 2026-02-24 10:30:00

-- 🟩 SQL Server (MSSQL)
SELECT GETDATE();           -- 결과: 2026-02-24 10:30:00.123
SELECT CURRENT_TIMESTAMP;   -- GETDATE()와 동일하게 작동
```

### ② 날짜 연산 (더하기/빼기) ️

특정 날짜에서 몇 달 뒤, 몇 일 전을 계산할 때 사용합니다. DB별로 문법 차이가 가장 심한 곳입니다.

**PostgreSQL: `INTERVAL` 키워드 사용**
- **작성법:** `날짜 + INTERVAL '숫자 단위'` 형태로 씁니다. 단위를 반드시 홑따옴표 안에 적어야 합니다
- **단위 종류:** `year`, `month`, `day`, `hour`, `minute`, `second` (복수형 `days`도 가능).빼고 싶다면 `- INTERVAL`을 쓰거나 숫자에 음수를 넣습니다.

**Oracle: 전용 함수 `ADD_MONTHS()` 사용**
- **문법:** `ADD_MONTHS(기준날짜, 더할_개월수)`
- 월(Month) 단위는 함수를 쓰지만, 일(Day) 단위는 그냥 날짜에 정수를 더하거나 빼면 됩니다.

**SQL Server (MSSQL): 만능 함수 `DATEADD()` 사용**
- **문법:** `DATEADD(날짜_단위, 더할_숫자, 기준날짜)`
- **특징:** 모든 연산(연, 월, 일, 시간)을 이 함수 하나로 다 처리합니다. 더할 숫자에 **음수(-)** 를 넣으면 과거 날짜(빼기)를 계산합니다.

```sql
-- 🐘 PostgreSQL
SELECT NOW() + INTERVAL '1 month';       -- 1달 뒤
SELECT NOW() - INTERVAL '1 year';        -- 1년 전
SELECT '2026-02-24'::DATE + 3;           -- 날짜에 바로 정수를 더하면 3일 뒤(Date) 반환

-- 🔴 Oracle (SQLD 단골)
SELECT ADD_MONTHS(SYSDATE, 1) FROM DUAL; -- 1달 뒤 
SELECT SYSDATE + 3 FROM DUAL;            -- 3일 뒤
SELECT SYSDATE + (1/24) FROM DUAL;       -- 1시간 뒤 (24시간을 나눠서 더함)
SELECT ADD_MONTHS(TO_DATE('2024-01-31'), 1) FROM DUAL; -- 결과: 2024-02-28 (2월엔 31일이 없으므로 마지막 날 반환)

-- 🟩 SQL Server (MSSQL) 실무 대장!
SELECT DATEADD(month, 1, GETDATE());     -- 1달 뒤
SELECT DATEADD(year, -1, GETDATE());     -- 1년 전 (음수 사용!)
SELECT DATEADD(day, 3, GETDATE());       -- 3일 뒤
SELECT DATEADD(hour, 1, GETDATE());      -- 1시간 뒤
```

#### [심화 꿀팁] 말일(End of Month) 계산의 치명적 차이 (Oracle vs 그 외)

'1월 31일 + 1달 = 2월 28/29일'처럼 도착하는 달에 해당 일자가 없으면 말일로 깎아주는 건 모든 DB가 동일합니다. 하지만 **'2월 28일(말일) + 1달'** 을 계산할 때 결과가 완전히 다릅니다! (SQLD 빈출 포인트)

**Oracle (`ADD_MONTHS`):** 
출발일이 그 달의 말일이었다면, 도착일도 무조건 **그 달의 말일(3월 31일)** 로 꽉 채워줍니다. (예: 2월 28일 + 1달 = 3월 31일 / 구독 서비스 결제일 갱신에 특화)
더해진 달에 기준 날짜의 '일(Day)'이 없으면 **그 달의 마지막 날**을 반환합니다. (예: 1월 31일 + 1달 = 2월 28/29일)

**PostgreSQL /  MSSQL**
딱 월(Month) 숫자만 바꿉니다. 출발일이 28일이었으므로, 도착일도 **3월 28일**이 됩니다.(예: 2월 28일 + 1달 = **3월 28일**)
더해진 달에 기준 날짜의 '일(Day)'이 없으면 **그 달의 마지막 날**을 반환합니다. (예: 1월 31일 + 1달 = 2월 28/29일)

※ 만약 Postgres나 MSSQL에서 무조건 말일을 구하고 싶다면, MSSQL은 `EOMONTH()` 함수를 쓰고, Postgres는 한 달을 더한 뒤 `DATE_TRUNC`로 1일로 만든 다음 하루를 빼는 꼼수(`- INTERVAL '1 day'`)를 써야 합니다.


### ③ 두 날짜의 간격 구하기 (Difference)

두 시점 사이에 얼마만큼의 시간이 흘렀는지 계산합니다. (나이 계산, D-Day 계산, 체류 시간 분석 등)

**PostgreSQL: `AGE()` 함수와 단순 뺄셈**
- **`AGE(최근날짜, 과거날짜)`:** 결과가 `INTERVAL` 타입으로 나옵니다. (예: 35년 8개월 26일)
- **단순 뺄셈 (`-`):** 두 날짜를 `DATE` 타입으로 빼면 순수하게 **'일(Day)' 차이**를 정수로 반환합니다.

 **Oracle: `MONTHS_BETWEEN()` 함수와 단순 뺄셈**
- **`MONTHS_BETWEEN(최근날짜, 과거날짜)`:** 두 날짜 사이의 **'개월 수'** 를 실수형으로 반환합니다. (예: 12.5개월)
- **단순 뺄셈 (`-`):** 날짜끼리 빼면 **'일(Day)' 차이**를 반환합니다.

**SQL Server (MSSQL): 만능 간격 계산기 `DATEDIFF()`**
- **문법:** `DATEDIFF(날짜_단위, 시작일(과거), 종료일(최근))`
- **특징:** 연, 월, 일, 시간, 초 등 원하는 단위로 간격을 딱 떨어지는 **정수**로 반환합니다.
- **주의사항:** `DATEADD`와 다르게 **과거 날짜가 먼저, 최근 날짜가 나중**에 들어갑니다! (순서가 바뀌면 음수가 나옵니다.)

```sql
-- 🐘 PostgreSQL
SELECT AGE('2026-02-24', '1990-05-01');           -- 반환: 35 years 9 mons 23 days
SELECT '2026-02-24'::DATE - '2026-02-20'::DATE;   -- 반환: 4 (일수 차이)

-- 🔴 Oracle (SQLD 단골)
SELECT MONTHS_BETWEEN(SYSDATE, '1990-05-01') FROM DUAL; -- 반환: 두 날짜 사이의 개월 수
SELECT SYSDATE - TO_DATE('2026-02-20') FROM DUAL;       -- 반환: 두 날짜 사이의 일수 차이

-- 🟩 SQL Server (MSSQL) 실무 대장!
-- 1990년생은 오늘(2026-02-24)까지 며칠을 살았을까?
SELECT DATEDIFF(day, '1990-05-01', GETDATE());    -- 반환: 일수 (정수)

-- 두 날짜 사이에 몇 '달'이 지났을까?
SELECT DATEDIFF(month, '2026-01-01', '2026-02-24'); -- 반환: 1 (개월 차이)

-- 두 시간 사이에 몇 '시간'이 지났을까?
SELECT DATEDIFF(hour, '2026-02-24 09:00:00', GETDATE()); -- 반환: 시간 차이 (정수)
```

### ④ 날짜 추출 (Extract) 

날짜 데이터에서 연도, 월, 일 등 **원하는 숫자 하나만 쏙 뽑아내어 정수(Number)로 반환**합니다

**표준 SQL (PostgreSQL, Oracle): `EXTRACT()`**
- **문법:** `EXTRACT(단위 FROM 날짜)`
- **단위:** `YEAR`, `MONTH`, `DAY`, `HOUR`. (Postgres는 `DOW`를 쓰면 요일을 0~6 숫자로 반환)

**SQL Server (MSSQL): `DATEPART()`**
- **문법:** `DATEPART(단위, 날짜)` 또는 단축 함수 `YEAR()`, `MONTH()` 사용.

```sql
-- 🐘 PostgreSQL / 🔴 Oracle (표준 SQL)
SELECT EXTRACT(YEAR FROM NOW());  -- 숫자 2026 반환
SELECT EXTRACT(MONTH FROM NOW()); -- 숫자 1 반환
SELECT EXTRACT(DOW FROM NOW());   -- Postgres 전용: 요일 반환 (0:일요일 ~ 6:토요일)

-- 🟩 SQL Server (MSSQL) 전용
SELECT DATEPART(YEAR, GETDATE());  -- 숫자 2026 반환
SELECT DATEPART(dw, GETDATE());    -- 요일 반환 (1:일요일 ~ 7:토요일, Postgres와 다름!)
SELECT YEAR(GETDATE());            -- 단축 함수 지원
```

### ⑤ 예쁘게 보여주기 (Formatting)

날짜 타입을 문자열(String) 타입으로 변환하면서 예쁜 포맷을 입혀줍니다.

**PostgreSQL / Oracle: `TO_CHAR()`**
- **문법:** `TO_CHAR(날짜, '포맷_문자열')`
- **주요 포맷:** `YYYY`(연도), `MM`(월), `DD`(일), `HH24`(24시간), `MI`(분), `SS`(초)

**SQL Server (MSSQL): `FORMAT()` & `CONVERT()`**
- **방식 1. `FORMAT(날짜, '포맷')`:** 가장 직관적인 최신 방식입니다. (주의: 월은 대문자 `MM`, 분은 소문자 `mm`을 씁니다.)
- **방식 2. `CONVERT(타입, 날짜, 스타일코드)`:** 실무 옛날 쿼리에서 엄청나게 많이 보입니다! 미리 정해진 **'숫자 코드'**를 외워야 하는 것이 특징입니다. (성능은 `FORMAT`보다 미세하게 더 빠릅니다.)

```sql
--  PostgreSQL /  Oracle
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD');  -- 문자열 '2026-01-27' 반환
SELECT TO_CHAR(NOW(), 'YYYY년 MM월'); -- 문자열 '2026년 01월' 반환
SELECT TO_CHAR(NOW(), 'Day');         -- 요일의 영문 이름 반환 (예: Tuesday)

--------------------------------------------------
-- SQL Server (MSSQL) 전용: FORMAT & CONVERT
--------------------------------------------------
-- 1. FORMAT (직관적이고 편리한 최신 방식)
SELECT FORMAT(GETDATE(), 'yyyy-MM-dd');  -- 결과: 2026-02-24
SELECT FORMAT(GETDATE(), 'yyyy년 MM월'); -- 결과: 2026년 02월

-- 2. CONVERT (실무에서 밥 먹듯이 보는 고전 방식!)
SELECT CONVERT(VARCHAR, GETDATE(), 23);  -- 결과: 2026-02-24 ( 코드 23: 가장 많이 씀!)
SELECT CONVERT(VARCHAR, GETDATE(), 112); -- 결과: 20260224   (코드 112: 파일명이나 로그 남길 때 씀)
SELECT CONVERT(VARCHAR, GETDATE(), 120); -- 결과: 2026-02-24 10:30:00 (코드 120: 시간까지 다 나옴)
```

---
## 상세 분석: `EXTRACT` vs `DATE_TRUNC` 차이점

초보자가 가장 많이 헷갈리는 부분이다. **"언제 뭘 써야 해?"**

| **함수**         | **결과물(Type)**   | **설명**                                  | **사용 예시 (PostgreSQL / Oracle)**                                                      |
| -------------- | --------------- | --------------------------------------- | ------------------------------------------------------------------------------------ |
| **`EXTRACT`**  | **숫자 (Number)** | 날짜에서 **부품 하나**만 떼어냄                     | "12월생 회원 찾기"<br><br>`WHERE EXTRACT(MONTH FROM date) = 12`                            |
| **`TRUNCATE`** | **날짜 (Date)**   | 날짜 형태를 유지하되, 지정 단위 **이하를 01일 00시로 초기화** | "**월별** 매출 집계하기"<br><br>🐘 `DATE_TRUNC('month', date)`<br><br>🔴 `TRUNC(date, 'MM')` |


**실무 핵심 (GROUP BY):**
월별 통계를 낼 때 `GROUP BY EXTRACT(MONTH FROM date)`를 해버리면, 2023년 1월과 2024년 1월이 똑같이 `1`로 합쳐져 버리는 대참사가 일어납니다! 
연/월을 안전하게 묶으려면 반드시 **`DATE_TRUNC` (또는 `TRUNC`)** 를 사용하세요

---
## 초보자가 자주 하는 실수 (Common Mistakes)

### ① "글자('2024-01-01')에다가 숫자를 더했어요."

- `'2024-01-01' + 1` 👉 에러! (글자에 숫자 못 더함)
- **해결:** `'2024-01-01'::DATE + 1` (날짜로 캐스팅 필수)

### ② "타임존(Timezone) 때문에 날짜가 달라져요."

- 서버 시간은 UTC(영국 기준)인데 한국(KST, +9)으로 생각하고 쿼리 짜면, 오전 9시 이전 데이터는 **'어제'**로 잡힘.
- **해결:** `NOW() AT TIME ZONE 'Asia/Seoul'` 처럼 타임존을 명시하거나, `DATE` 타입으로 변환해서 사용.

### ③ "두 날짜 뺐는데 이상한 숫자가 나와요."

- `TIMESTAMP - TIMESTAMP` = **`INTERVAL`** (예: `3 days 04:00:00`)
- `DATE - DATE` = **`INTEGER`** (예: `3`)
- D-Day 계산할 거면 둘 다 `::DATE`로 맞춰서 빼는 게 정신건강에 좋음.