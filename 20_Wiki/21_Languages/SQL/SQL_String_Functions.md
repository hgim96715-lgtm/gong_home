---
aliases:
  - SQL 문자열 함수
  - 문자열 조작
  - SUBSTR
  - LTRIM
  - REPLACE
  - SPLIT
  - STRING_AGG
  - GROUP_CONCAT
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Numeric_Functions]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Type_Casting]]"
---
# SQL 문자열 함수 — 자르고 붙이고 청소하기

## 개념 한 줄 요약

> **"사용자가 입력한 데이터는 생각보다 훨씬 더럽다. 띄어쓰기 오류·대소문자 혼용 등을 정제(Cleansing)하는 데이터 엔지니어의 필수 도구들."**

---

## 왜 필요한가?

현업의 Raw 데이터는 완벽하지 않다. 이메일 대소문자가 섞여 있거나, 쉼표로 뭉쳐 있는 등 포맷이 파편화되어 있기 때문에, 정제해야 정확한 분석과 JOIN 이 가능하다.

| 목적                   | 함수                             |
| -------------------- | ------------------------------ |
| 데이터 정제 (공백·특수문자 제거)  | `TRIM` · `REPLACE`             |
| 특정 자리 추출 (주민번호·전화번호) | `SUBSTR`                       |
| 대소문자 통일              | `UPPER` · `LOWER`              |
| 컬럼 쪼개기               | `SPLIT_PART` · `REGEXP_SUBSTR` |
| 여러 행을 한 줄로 합치기       | `STRING_AGG` · `LISTAGG`       |
| 자릿수 맞추기              | `LPAD` · `RPAD`                |

---

---

# ① 대소문자 통일 — UPPER · LOWER

이메일이나 아이디 비교 시 양쪽을 통일해야 누락이 발생하지 않는다.

```sql
SELECT UPPER('sql')  FROM DUAL;  -- 결과: 'SQL'
SELECT LOWER('SQL')  FROM DUAL;  -- 결과: 'sql'

-- 실무 예: 대소문자 관계없이 이메일 비교
WHERE LOWER(email) = LOWER('User@Gmail.com')
```

---

---

# ② TRIM — 공백·특정 문자 제거

## 기본 문법

|함수|방향|지울 대상|
|---|---|---|
|`LTRIM(문자열, [지울문자])`|왼쪽|공백 또는 지정 문자|
|`RTRIM(문자열, [지울문자])`|오른쪽|공백 또는 지정 문자|
|`TRIM([방향] [지울문자] FROM 문자열)`|양쪽(기본)|공백 또는 지정 문자|

```sql
-- 공백 제거 (모든 DB 공통)
SELECT LTRIM('   SQLD') FROM DUAL;              -- 결과: 'SQLD'
SELECT RTRIM('SQLD   ') FROM DUAL;              -- 결과: 'SQLD'
SELECT TRIM('   SQLD   ') FROM DUAL;            -- 결과: 'SQLD'

-- TRIM 방향 지정 (표준 문법)
SELECT TRIM(LEADING  'x' FROM 'xxSQLxx');       -- 결과: 'SQLxx'  (왼쪽만)
SELECT TRIM(TRAILING 'x' FROM 'xxSQLxx');       -- 결과: 'xxSQL'  (오른쪽만)
SELECT TRIM(BOTH     'x' FROM 'xxSQLxx');       -- 결과: 'SQL'    (양쪽)
```

---

## CHAR 타입과 RTRIM — SQLD 빈출 ⭐️

> **`CHAR(n)` 타입은 데이터 길이가 n 보다 짧으면 나머지를 자동으로 공백으로 채운다.** 따라서 `RTRIM` 을 쓰면 그 뒤쪽 공백이 제거된다.

```
COL2 컬럼 타입: CHAR(5)
저장된 값: 'SQL'

실제 저장 상태: 'SQL  '  ← 3글자 + 공백 2칸 (총 5자리 고정)
                       ↑↑
                       CHAR 타입이 자동으로 채운 공백
```

```sql
-- CHAR(5) 컬럼에 'SQL' 이 저장된 경우

SELECT LENGTH(col2) FROM 테이블;          -- 결과: 5  (공백 포함 5자리)
SELECT LENGTH(RTRIM(col2)) FROM 테이블;   -- 결과: 3  (공백 제거 후 실제 값만)

SELECT RTRIM(col2) FROM 테이블;           -- 결과: 'SQL' (뒤쪽 공백 2칸 제거)
SELECT col2 = 'SQL' FROM 테이블;          -- 결과: DB 마다 다름 (공백 포함 여부)
```

> **왜 중요한가?** `CHAR` 타입 컬럼끼리 `{text}=` 비교할 때 공백 때문에 예상과 다른 결과가 나올 수 있다. 
> `RTRIM` 으로 공백을 제거한 뒤 비교하거나, 처음부터 `VARCHAR` 를 쓰는 것이 안전하다.

```sql
-- CHAR 타입 비교 시 공백 함정
WHERE col2 = 'SQL'           -- ⚠️ DB 따라 'SQL  ' 과 'SQL' 이 다를 수 있음
WHERE RTRIM(col2) = 'SQL'    -- ✅ 공백 제거 후 비교 → 안전
```

---

## DB 별 TRIM 차이 — SQL Server 의 배신 ⚠️

|DB|`LTRIM` / `RTRIM` 특정 문자 지정|
|---|---|
|Oracle · PostgreSQL|✅ 가능 (`LTRIM('xxxSQLD', 'x')`)|
|SQL Server|❌ 공백만 제거 가능. 특정 문자 지정 시 에러|

```sql
-- ✅ Oracle / PostgreSQL: 특정 문자 제거
SELECT LTRIM('xxxSQLD', 'x') FROM DUAL;   -- 결과: 'SQLD'

-- ❌ SQL Server: 에러!
SELECT LTRIM('xxxSQLD', 'x');             -- 인자 1개만 허용
```

---

## TRIM 의 두 가지 핵심 법칙

### 법칙 1 — 만나면 즉시 멈춘다 (연속성의 법칙)

지정하지 않은 문자를 만나는 순간 그 자리에서 작업을 종료한다. 중간에 있는 대상 문자는 건너뛰지 못하고 살아남는다.

```sql
SELECT RTRIM('DBSQLxxYxx', 'x') FROM DUAL;
-- 오른쪽에서 xx 제거 → 'Y' 를 만나는 순간 멈춤
-- 결과: 'DBSQLxxY'  (중간의 xx 는 살아남음)
```

### 법칙 2 — 단어가 아니라 낱개 문자다

두 번째 인자 `'SQL'` 은 `"SQL"` 이라는 단어를 찾는 게 아니다. `'S'` 또는 `'Q'` 또는 `'L'` 중 하나라도 나오면 순서 관계없이 지운다.

```sql
SELECT RTRIM('DBSLQQSS', 'SQL') FROM DUAL;
-- 오른쪽에서 S, S, Q, Q, L, S 를 낱개로 하나씩 제거
-- 'B' 를 만나는 순간 멈춤
-- 결과: 'DB'
```

---

---

# ③ SUBSTR — 문자열 자르기

주민등록번호에서 성별 자리 추출, 전화번호 앞자리 추출 등에 사용한다.

```
문법: SUBSTR(문자열, 시작위치, [자를길이])
길이 생략 시 → 시작위치부터 끝까지 전부 가져옴
```

```sql
SELECT SUBSTR('SQLD', 2, 2)  FROM DUAL;  -- 결과: 'QL' (2번째부터 2개)
SELECT SUBSTR('SQLD', 2)     FROM DUAL;  -- 결과: 'QLD' (2번째부터 끝까지)
SELECT SUBSTR('SQLD', -2, 2) FROM DUAL;  -- 결과: 'LD' (뒤에서 2번째부터 2개, Oracle 전용)
```

## DB 별 차이

|DB|함수명|음수 인덱스|비고|
|---|---|---|---|
|Oracle|`SUBSTR()`|✅ 지원|뒤에서 N번째|
|PostgreSQL|`SUBSTR()` · `SUBSTRING()` 둘 다|❌ 미지원|`RIGHT()` 사용 권장|
|SQL Server|`SUBSTRING()`|❌ 미지원|함수명 다름!|

```sql
-- PostgreSQL 에서 뒤에서 자르기 (실무 국룰)
SELECT RIGHT('SQLD', 2);    -- 결과: 'LD' (뒤에서 2글자)
SELECT LEFT('SQLD', 2);     -- 결과: 'SQ' (앞에서 2글자)
```

---

---

# ④ LENGTH — 문자열 길이 재기

```sql
SELECT LENGTH('SQLD') FROM DUAL;   -- 결과: 4
```

## DB 별 함수명 차이

|DB|함수|예시|결과|
|---|---|---|---|
|Oracle|`LENGTH()`|`LENGTH('SQLD')`|`4`|
|PostgreSQL|`LENGTH()` · `CHAR_LENGTH()`|동일|`4`|
|SQL Server|`LEN()`|`LEN('SQLD')`|`4`|

> **SQL Server 는 `LEN`** (끝에 `GTH` 없음). SQLD 시험 함정.

```sql
-- ⚠️ 공백 문자도 1글자로 카운트된다
SELECT LENGTH('SQL' || CHR(10) || 'D') FROM DUAL;  -- 결과: 5 (\n 포함)
-- 줄바꿈(\n), 탭(\t) 같은 공백 문자도 1글자 취급
```

---

---

# ⑤ REPLACE — 찾아서 바꾸기

```
문법: REPLACE(문자열, 찾을문자, 바꿀문자)
```

```sql
-- 치환
SELECT REPLACE('010-1234-5678', '-', '*');   -- 결과: '010*1234*5678'

-- 삭제 (Oracle: 3번째 인자 생략 가능)
SELECT REPLACE('010-1234-5678', '-');        -- 결과: '01012345678' (Oracle 전용)

-- 삭제 (PostgreSQL · SQL Server: 빈 문자열 '' 명시 필수)
SELECT REPLACE('010-1234-5678', '-', '');    -- 결과: '01012345678'
```

## DB 별 3번째 인자 생략 여부 — SQLD 빈출 ⭐️

|DB|3번째 인자 생략|결과|
|---|---|---|
|Oracle|✅ 생략 가능|해당 문자 완전 삭제|
|PostgreSQL|❌ 생략 불가|에러 → `''` 명시 필요|
|SQL Server|❌ 생략 불가|에러 → `''` 명시 필요|

## CHR(10) 응용 — 줄바꿈 제거

```sql
-- CHR(10) = ASCII 10번 = 줄바꿈 문자(\n)
SELECT REPLACE(col1, CHR(10))      -- 줄바꿈 삭제 (Oracle 전용, 3번째 인자 생략)
SELECT REPLACE(col1, CHR(10), '')  -- 줄바꿈 삭제 (PostgreSQL · SQL Server)
```

---

---

# ⑥ LPAD · RPAD — 빈자리 채우기

```
문법: LPAD(문자열, 총길이, [채울문자])
```

```sql
SELECT LPAD('123', 5, '0');   -- 결과: '00123' (왼쪽에 0 채움)
SELECT RPAD('AB', 5, '*');    -- 결과: 'AB***' (오른쪽에 * 채움)
```

> 상품 코드·사원번호의 자릿수를 0으로 맞출 때 주로 사용한다.

---

---

# ⑦ SPLIT — 문자열 쪼개기

하나의 컬럼에 구분자(`,`, `@`)로 뭉쳐있는 데이터를 쪼갤 때 사용한다. 표준 SQL 이 아니어서 **DB별 문법 차이가 가장 심한 영역**이다.

## SPLIT_PART — PostgreSQL ⭐️

```text
SPLIT_PART(문자열, 구분자, N번째)
  문자열을 구분자로 쪼갠 후 N번째 조각 반환
  1번째부터 시작 (0 아님)
  PostgreSQL 전용

예) SPLIT_PART('지연 도착(+15분)', '(', N)

  '지연 도착(+15분)'
       ↓ '(' 로 쪼개면
  1번째: '지연 도착'
  2번째: '+15분)'

  N=1 → '지연 도착'   ← 괄호 앞
  N=2 → '+15분)'      ← 괄호 뒤
```

```sql
-- 🐘 PostgreSQL: SPLIT_PART (가장 직관적)
-- 문법: SPLIT_PART(문자열, 구분자, N번째)
SELECT SPLIT_PART(이메일, '@', 2) AS 도메인 FROM 사원테이블;
-- 결과: 'gmail.com', 'naver.com'

-- 실전: 괄호 앞 텍스트만 추출
SELECT SPLIT_PART('정시 도착(기준 시각)', '(', 1);   -- '정시 도착'
SELECT SPLIT_PART(arr_status, '(', 1) FROM train_delay;  -- 상태 텍스트만
-- arr_status = '지연 도착(+15분)' → '지연 도착'
-- arr_status = '정시 도착(기준 시각)' → '정시 도착'

-- 🔴 Oracle: REGEXP_SUBSTR (정규식 조합)
-- '[^@]+': @ 가 아닌 문자 덩어리 중 2번째 덩어리
SELECT REGEXP_SUBSTR(이메일, '[^@]+', 1, 2) AS 도메인 FROM 사원테이블;

-- SQL Server: STRING_SPLIT (세로로 분리됨)
SELECT 사원명, value AS 개별_취미
FROM 사원테이블
CROSS APPLY STRING_SPLIT(취미목록, ',');
-- 결과: 옆으로 컬럼이 늘어나는 게 아니라 행(Row) 이 아래로 여러 개 생성
-- 홍길동 | 독서
-- 홍길동 | 등산
-- 홍길동 | 게임
```

---

---

# ⑧ STRING_AGG — 여러 행을 하나로 합치기

> **SPLIT 의 정반대.** `GROUP BY` 로 묶인 여러 행을 구분자로 이어서 하나의 문자열로 만든다.

## DB 별 문법 비교

|DB|함수|정렬 위치|
|---|---|---|
|PostgreSQL|`STRING_AGG(컬럼, '구분자' ORDER BY ...)`|괄호 **안**|
|MySQL|`GROUP_CONCAT(컬럼 ORDER BY ... SEPARATOR '구분자')`|괄호 **안**|
|Oracle|`LISTAGG(컬럼, '구분자') WITHIN GROUP (ORDER BY ...)`|괄호 **밖**|
|SQL Server|`STRING_AGG(컬럼, '구분자') WITHIN GROUP (ORDER BY ...)`|괄호 **밖**|

```sql
-- 🐘 PostgreSQL
SELECT 부서명,
       STRING_AGG(DISTINCT 사원명, ', ' ORDER BY 사원명 ASC) AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;
-- 결과: 영업부 | '김철수, 이영희, 홍길동'

-- MySQL
SELECT 부서명,
       GROUP_CONCAT(DISTINCT 사원명 ORDER BY 사원명 ASC SEPARATOR ', ') AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;

-- 🔴 Oracle
SELECT 부서명,
       LISTAGG(사원명, ', ') WITHIN GROUP (ORDER BY 사원명 ASC) AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;

-- SQL Server (2017+)
SELECT 부서명,
       STRING_AGG(사원명, ', ') WITHIN GROUP (ORDER BY 사원명 ASC) AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;
```

> PostgreSQL · MySQL → 정렬이 괄호 **안** Oracle · SQL Server → `WITHIN GROUP` 으로 괄호 **밖**에서 정렬

---

---

# ⑨ CHR · ASCII — 문자 ↔ 코드 변환

```sql
-- 코드 → 문자
SELECT CHR(65)   FROM DUAL;   -- 결과: 'A'  (Oracle · PostgreSQL 공통)
SELECT CHAR(65);               -- 결과: 'A'  (SQL Server)

-- 문자 → 코드 (세 DB 모두 공통)
SELECT ASCII('A');             -- 결과: 65
```

|DB|코드→문자 함수|문자→코드 함수|
|---|---|---|
|Oracle|`CHR()`|`ASCII()`|
|PostgreSQL|`CHR()`|`ASCII()`|
|SQL Server|`CHAR()`|`ASCII()`|

---

---
