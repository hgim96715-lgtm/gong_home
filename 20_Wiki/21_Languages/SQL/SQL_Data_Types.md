---
aliases:
  - 자료형
  - 데이터타입
  - cast
  - casting
  - varchar
  - integer
  - timestamp
  - boolean
  - JSONB
  - JSON
  - INDEX
tags:
  - SQL
related:
  - "[[SQL_Window_Functions]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DDL_Create]]"
  - "[[SQL_Type_Casting]]"
---

# SQL 데이터 타입 (Data Types)

## 개념 한 줄 요약

> **"데이터를 담는 그릇의 모양을 정하는 규칙."** 숫자는 Number 그릇, 글자는 String 그릇, 날짜는 Date 그릇에 담아야 컴퓨터가 오해 없이 처리한다.

---

## 왜 필요한가?

```
문제 ①: '100' + '200' = '100200'  (숫자가 아닌 글자로 인식 → 이어붙임)
문제 ②: '2024-01-01' - 1일 = 에러  (글자에서 숫자를 뺄 수 없음)

해결: 컬럼 생성 시 타입을 못 박아두거나, 필요할 때 형변환(Casting) 으로 해결
```

## 실무 타입 선택 기준

|데이터 종류|타입|이유|
|---|---|---|
|이름, 주소|`VARCHAR`|길이가 다양함|
|**전화번호**|`VARCHAR` ⭐️|010 → 숫자로 저장하면 10이 되어버림|
|가격, 수량, 나이|`INTEGER` / `NUMERIC`|계산이 필요한 숫자|
|가입일, 결제시간|`TIMESTAMP`|시간까지 기록 필요|
|탈퇴 여부|`BOOLEAN`|TRUE / FALSE|

---

---

# ① SQLD vs PostgreSQL 타입 비교

> **SQLD 시험은 Oracle 기준.** 시험엔 `VARCHAR2`, `NUMBER` 를, 실무엔 `VARCHAR`, `INTEGER`, `TEXT` 를 쓴다.

| 종류        | SQLD / Oracle | PostgreSQL           | 설명                            |
| --------- | ------------- | -------------------- | ----------------------------- |
| 고정 문자     | `CHAR(n)`     | `CHAR(n)`            | n자리 고정. 남으면 공백으로 채움           |
| 가변 문자     | `VARCHAR2(n)` | `VARCHAR(n)`         | 입력한 만큼만 저장                    |
| 제한없는 문자   | `CLOB`        | `TEXT`               | PostgreSQL TEXT 는 크기 제한 없음 ⭐️ |
| 정수        | `NUMBER(n)`   | `INTEGER` / `BIGINT` | PostgreSQL 은 정수 전용 타입 있음      |
| 소수        | `NUMBER(p,s)` | `NUMERIC(p,s)`       | p: 전체 자릿수, s: 소수점 자릿수         |
| 날짜만       | `DATE`        | `DATE`               | 년-월-일만 저장                     |
| 날짜+시간     | `TIMESTAMP`   | `TIMESTAMP`          | 시간까지 저장                       |
| 날짜+시간+타임존 | 없음            | `TIMESTAMPTZ`        | 글로벌 서비스라면 무조건 이거 ⭐️           |
| 고유 식별자    | 없음            | `UUID`               | 분산환경 PK 로 최적 ⭐️               |
| 논리형       | 없음            | `BOOLEAN`            | TRUE / FALSE                  |

---

---

# ② PostgreSQL 필수 타입 4대장

## 문자형 (Character)

|타입|특징|언제 쓰나|
|---|---|---|
|`VARCHAR(n)`|최대 n글자, 입력한 만큼만 저장|대부분의 문자 데이터|
|`TEXT`|길이 제한 없음, VARCHAR 과 성능 차이 없음 ⭐️|긴 텍스트, 실무 애용|
|`CHAR(n)`|무조건 n글자, 남는 공간 공백으로 채움|코드값 등 길이 고정된 데이터|

### VARCHAR vs CHAR — 공백 비교 방식 차이 ⭐️

```sql
-- CHAR(10) 에 'ABC' 저장 → 내부적으로 'ABC       ' (7칸 공백 자동 채움)

-- CHAR 비교: 공백 무시 → 같은 값으로 인정
WHERE char_col = 'ABC'        -- 'ABC       ' = 'ABC'  → TRUE  ✅
WHERE char_col = 'ABC       ' -- 'ABC       ' = 'ABC  ' → TRUE  ✅

-- VARCHAR 비교: 공백 포함 → 다른 값으로 취급
WHERE varchar_col = 'ABC'     -- 'ABC ' != 'ABC'   → FALSE ❌
WHERE varchar_col = 'ABC '    -- 'ABC ' = 'ABC '   → TRUE  ✅
```

|타입|`'ABC'` vs `'ABC '`|이유|
|---|:-:|---|
|`CHAR`|**같음** ✅|공백으로 채우는 구조라 비교 시 공백 무시|
|`VARCHAR`|**다름** ❌|있는 그대로 저장 → 공백도 데이터|

> **SQLD 오답 유형:** `CHAR(10)` 에 `'A'` 저장 후
>  `WHERE col = 'A'` → **TRUE (정답)** `WHERE col = 'A '` 도 → **TRUE (정답)** 
>  `VARCHAR(10)` 에 `'A '` 저장 후 `WHERE col = 'A'` → **FALSE (공백 다름)**

---

## 숫자형 (Numeric)

|타입|특징|언제 쓰나|
|---|---|---|
|`INTEGER` (INT)|정수, 소수점 없음|개수, 나이, 등수|
|`NUMERIC` / `DECIMAL`|소수점 있는 **정확한** 숫자 ⭐️|**돈(Money) 계산 필수**|
|`FLOAT` / `REAL`|소수점 있는 **근사치**|과학 계산용 (금융 데이터 ❌)|

> **`FLOAT` 을 금융 데이터에 쓰면 안 되는 이유:** 부동소수점 특성상 `0.1 + 0.2 = 0.30000000000000004` 같은 미세한 오차가 발생한다. 돈 계산은 반드시 `NUMERIC` 사용.

---

## 날짜형 (Date/Time)

|타입|저장 형태|특징|
|---|---|---|
|`DATE`|`2024-01-27`|년-월-일만|
|`TIMESTAMP`|`2024-01-27 10:30:00`|시간까지|
|`TIMESTAMPTZ`|`2024-01-27 10:30:00+09`|타임존 포함 ⭐️ 글로벌 서비스 필수|

```sql
-- DATE 끼리 빼면 → Integer (일수)
SELECT '2024-01-27'::DATE - '2024-01-24'::DATE;  -- 결과: 3

-- TIMESTAMP 끼리 빼면 → Interval (시간 간격)
SELECT NOW() - '2024-01-01'::TIMESTAMP;  -- 결과: '26 days 10:30:00'

-- D-Day 구할 때는 ::date 로 변환 후 빼야 깔끔한 숫자
SELECT '2024-12-31'::DATE - NOW()::DATE AS d_day;
```

---

## 논리형 (Boolean)

```sql
-- TRUE 로 인식되는 값
TRUE, 1, 'yes', 'on', 't'

-- FALSE 로 인식되는 값
FALSE, 0, 'no', 'off', 'f'
```

---

## 특수 식별자 — UUID

> 128비트의 전 세계 유일한 식별자. 형식: `aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee`

```
VARCHAR(36) 로 저장하는 것보다:
저장 공간 절반 이하 (16 byte)
검색 속도 빠름
분산 환경(MSA, Kafka) PK 로 최적 ← PostgreSQL 만의 특권
```

---

---

# ③ 형변환 (Casting)

> **"데이터의 타입을 강제로 바꾸는 기술."**

## 표준 문법 — CAST (모든 DB 공통)

```sql
SELECT CAST('123' AS INTEGER);        -- 문자 → 숫자
SELECT CAST('2024-01-27' AS DATE);    -- 문자 → 날짜
SELECT CAST(99.9 AS INTEGER);         -- 소수 → 정수 (반올림)
```

## PostgreSQL 전용 — `::` (실무 애용) ⭐️

```sql
SELECT '123'::INTEGER + 1;            -- 결과: 124
SELECT '2024-01-27'::DATE - 1;        -- 결과: 2024-01-26
SELECT 99.9::INTEGER;                 -- 결과: 100 (반올림)
SELECT NOW()::DATE;                   -- 결과: 오늘 날짜만 (시간 제거)
```

> `CAST('123' AS INTEGER)` = `'123'::INTEGER` → 완전히 동일한 동작. PostgreSQL 실무에서는 `::` 가 훨씬 짧아서 선호.

---

---

# 초보자 실수 체크리스트

## "전화번호는 숫자니까 INTEGER 로 해야지"

```
❌ 절대 안 됨
01012345678 → INTEGER 로 저장 → 1012345678 (앞 0 사라짐!)

계산을 안 하는 숫자 = 문자로 저장
전화번호, 주민번호, 우편번호, 카드번호 → 전부 VARCHAR
```

## "NULL 이랑 빈 문자('') 는 같은 거 아닌가요?"

```
NULL  = "데이터가 없음" (부재중)
''    = "내용이 비어있음" (빈 상자가 있음)

IS NULL 로 조회할 때 '' 는 검색되지 않음
```

```sql
WHERE col IS NULL     -- NULL 만 검색 ('' 는 안 나옴)
WHERE col = ''        -- 빈 문자열만 검색 (NULL 은 안 나옴)
WHERE col IS NULL OR col = ''  -- 둘 다 잡으려면 OR 사용
```

## "날짜 계산 결과가 이상해요"

```sql
-- DATE - DATE  → 정수 (일수)
'2024-01-27'::DATE - '2024-01-24'::DATE   -- 결과: 3

-- TIMESTAMP - TIMESTAMP  → Interval
NOW() - '2024-01-01'::TIMESTAMP           -- 결과: '26 days 10:30:00'

-- D-Day 구할 때 타입 통일 필수
'2024-12-31'::DATE - NOW()::DATE          -- 결과: 정수 (일수)
```

## "FLOAT 으로 금융 계산했더니 값이 이상해요"

```sql
SELECT 0.1::FLOAT + 0.2::FLOAT;   -- 결과: 0.30000000000000004 💥
SELECT 0.1::NUMERIC + 0.2::NUMERIC; -- 결과: 0.3 ✅

-- 금융 데이터는 무조건 NUMERIC / DECIMAL
```

---
---
# ④ JSONB — JSON 데이터 저장 타입

## 한 줄 요약

> **"JSON 을 파싱해서 바이너리로 저장하는 타입. 검색·인덱싱이 가능해서 TEXT 에 JSON 넣는 것보다 훨씬 강력하다."**

---

## JSON vs JSONB 차이

|항목|`JSON`|`JSONB`|
|---|---|---|
|저장 방식|텍스트 그대로|파싱 후 바이너리로|
|쓰기 속도|빠름|약간 느림 (파싱 비용)|
|읽기 / 검색|느림|**빠름** ⭐️|
|인덱스|❌ 불가|✅ 가능|
|공백 / 키 순서|원문 그대로 보존|❌ 정규화됨 (순서 바뀔 수 있음)|
|실무 선택|거의 안 씀|**거의 항상 JSONB** ⭐️|

```
JSON 을 쓰는 경우:
  원문 그대로 보존이 필수일 때 (로그 포맷 등)
  쓰기만 하고 검색은 거의 없을 때

JSONB 를 쓰는 경우:
  저장 후 특정 키로 검색·필터가 필요할 때
  인덱스를 걸어서 성능을 높이고 싶을 때
  → 대부분의 실무 케이스 = JSONB
```

---

## JSONB 연산자

```sql
-- 예시 데이터
-- prices_raw = '[{"name": "성인", "price": 23000}, {"name": "어린이", "price": 18000}]'

-- → 특정 인덱스 요소 접근 (배열)
SELECT prices_raw -> 0                  FROM raw_exhibitions;
-- 결과: {"name": "성인", "price": 23000}   (JSON 타입)

-- ->> 텍스트로 꺼내기
SELECT prices_raw -> 0 ->> 'name'       FROM raw_exhibitions;
-- 결과: 성인  (TEXT 타입)

SELECT prices_raw -> 0 ->> 'price'      FROM raw_exhibitions;
-- 결과: 23000 (TEXT — 숫자 쓰려면 ::INTEGER 변환 필요)

-- @> 포함 여부 확인 (필터 조건으로 활용)
SELECT * FROM raw_exhibitions
WHERE prices_raw @> '[{"name": "성인"}]';
-- 성인 가격이 있는 전시만 조회
```

|연산자|의미|반환 타입|
|---|---|---|
|`->`|키 / 인덱스로 요소 접근|JSONB|
|`-->`|키 / 인덱스로 요소 접근 (텍스트 반환)|TEXT|
|`@>`|왼쪽이 오른쪽을 포함하는가?|BOOLEAN|
|`?`|해당 키가 존재하는가?|BOOLEAN|

---

## JSONB 에 인덱스 걸기

```sql
-- GIN 인덱스 — JSONB 전용 (포함 검색 @> 에 효과적)
CREATE INDEX idx_prices_gin
    ON raw_exhibitions USING GIN (prices_raw);

-- 이후 @> 조건 쿼리가 빠르게 동작
SELECT * FROM raw_exhibitions
WHERE prices_raw @> '[{"name": "성인"}]';
```

---

## 실전 패턴 — dbt 파싱 전 원문 보관

```sql
-- 크롤러: API 응답 JSON 을 그대로 넣음
INSERT INTO raw_exhibitions (exhibition_id, prices_raw)
VALUES ('26002594', '[{"name":"성인","price":23000},{"name":"어린이","price":18000}]'::JSONB);

-- dbt staging 에서 파싱
SELECT
    exhibition_id,
    (elem ->> 'name')::TEXT    AS price_name,
    (elem ->> 'price')::INTEGER AS price_amount
FROM raw_exhibitions,
     LATERAL jsonb_array_elements(prices_raw) AS elem;
```

> `TEXT` 에 JSON 문자열로 저장하면 위 연산자와 `jsonb_array_elements` 를 쓸 수 없음 → 반드시 `JSONB` 타입으로 선언

---

---

# ⑤ INDEX — 컬럼에 목차 달기

## 한 줄 요약

> **"특정 컬럼에 미리 정렬된 주소록을 만들어두어 조회 속도를 높이는 구조. 데이터 타입은 아니지만 컬럼 설계 시 함께 결정한다."**

---

## 인덱스가 필요한 타입 vs 불필요한 타입

```
인덱스 효과가 큰 타입:
  VARCHAR / TEXT — 문자열 검색 (WHERE name = '...' / LIKE '...')
  DATE / TIMESTAMP — 날짜 범위 조회 (BETWEEN / >= / <=)
  INTEGER / NUMERIC — 숫자 범위 / 정렬

인덱스 효과가 작은 타입:
  BOOLEAN — 값이 TRUE/FALSE 두 개뿐
             → 테이블 절반을 스캔하므로 효과 미미
             → Partial Index 로 보완 (WHERE is_active = TRUE)
  JSONB — 일반 B-tree 인덱스 불가
          → GIN 인덱스 (jsonb 전용) 사용해야 함
```

---

## 타입별 인덱스 선택

| 타입                    | 인덱스 종류     | 비고                             |
| --------------------- | ---------- | ------------------------------ |
| `VARCHAR` / `TEXT`    | B-tree     | 기본 (LIKE '앞%' 만 효과, '%앞' 은 ❌)  |
| `INTEGER` / `NUMERIC` | B-tree     | 범위 조건에 효과적                     |
| `DATE` / `TIMESTAMP`  | B-tree     | 날짜 범위 조건에 효과적                  |
| `JSONB`               | **GIN**    | `@>` 포함 검색에 사용                 |
| `TEXT` (전문 검색)        | **GIN**    | `tsvector` 기반 전문 검색            |
| `BOOLEAN`             | Partial 권장 | `WHERE col = TRUE` 로 범위 좁혀서 생성 |

```sql
-- VARCHAR 에 기본 B-tree
CREATE INDEX idx_ex_location ON raw_exhibitions (location);

-- JSONB 에 GIN 인덱스
CREATE INDEX idx_prices_gin ON raw_exhibitions USING GIN (prices_raw);

-- BOOLEAN 에 Partial Index
CREATE INDEX idx_ex_active ON raw_exhibitions (start_date)
WHERE is_active = TRUE;
```

> 인덱스 상세 → [[PostgreSQL_Setup#⑦ INDEX — 검색 속도 높이기]] 참고

---
