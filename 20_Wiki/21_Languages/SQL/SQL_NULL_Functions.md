---
aliases:
  - NULL 함수
  - NVL
  - COALESCE
  - ISNULL
  - NULLIF
tags:
  - SQL
related:
  - "[[SQL_Understanding_NULL]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_Type_Casting]]"
  - "[[00_SQL_HomePage]]"
---
# NULL 관련 함수

## 개념 한 줄 요약 (Concept Summary)

**"모든 연산을 집어삼키는 블랙홀(NULL)을 안전한 숫자로 메워주는 심폐소생술."** 
데이터베이스에서 `NULL`은 '0'이나 '빈 칸(공백)'이 아닙니다. **"아직 모름(Unknown)"** 또는 **"존재하지 않음"** 을 의미하는 특수한 상태입니다. 
이 녀석을 방치하면 통계가 망가지고 쿼리 에러가 터집니다.

---
## 왜 필요한가? (Why)

- **블랙홀 연산 방지:** `100 + NULL = NULL` 입니다. 보너스 포인트가 NULL인 회원의 총 포인트를 계산할 때, 전체 포인트가 날아가는 대참사를 막아야 합니다.
- **통계 왜곡 방지:** `AVG()` 같은 집계 함수는 NULL을 분모에서 아예 빼버립니다. 0점 처리해야 할 유저가 계산에서 빠지면 평균이 비정상적으로 높아집니다.
- **UI/UX 개선:** 화면에 `null`이라는 글자가 그대로 노출되는 것을 막고, `미입력`이나 `0원`으로 예쁘게 바꿔 보여주기 위해 사용합니다.

---
## 핵심 함수 4대장

### ① 만국 공통어: `COALESCE`

**문법:** `COALESCE(값1, 값2, 값3, ... , 최후의_보루)`
**특징:** **표준 SQL**입니다. 나열된 값들 중에서 **'가장 처음으로 NULL이 아닌 값'** 을 반환합니다. 인자를 무한대로 넣을 수 있는 것이 엄청난 장점입니다!

```sql
-- 🐘 Postgres / 🔴 Oracle / 🟩 MSSQL 모두 지원!
SELECT COALESCE(NULL, NULL, 'C', 'D');   -- 결과: 'C'

-- 실무 활용: 핸드폰번호 없으면 집전화, 그것도 없으면 '연락처 없음' 출력
SELECT COALESCE(phone_num, home_num, '연락처 없음') FROM users;
```

### ② 2개만 콕 집어 교체: `NVL` / `ISNULL`

**특징:** 딱 2개의 인자만 받습니다. `(검사할_컬럼, NULL일_때_대체할_값)`

**반환값 (핵심 로직!):** 
* 첫 번째 인자(컬럼)가 **NULL이 아니면** 👉 원래 자신의 값을 그대로 반환합니다.
- 첫 번째 인자(컬럼)가 **NULL이면** 👉 두 번째 인자(대체할 값)를 반환합니다.

**DB별 차이 (매우 중요!):**
- **Oracle:** `NVL()` 을 씁니다. (Null VaLue의 약자)
- **SQL Server (MSSQL):** `ISNULL()` 을 씁니다.
- **PostgreSQL:** 이런 전용 함수 없이 그냥 무조건 표준인 `COALESCE()`를 씁니다.

```sql
-- 🔴 Oracle
-- bonus가 500이면 500 반환, NULL이면 0 반환!
SELECT NVL(bonus, 0) FROM salary;       

-- 🟩 SQL Server (MSSQL)
-- name이 '홍길동'이면 '홍길동' 반환, NULL이면 '무명' 반환!
SELECT ISNULL(name, '무명') FROM users;
```

### ③ 똑같으면 NULL로 폭파!: `NULLIF`

- **문법:** `NULLIF(값1, 값2)`
- **특징:** 두 값이 **같으면 `NULL`** 을 반환하고, **다르면 첫 번째 값(값1)** 을 반환합니다.
- **용도:** 실무에서 **'0으로 나누기(Divide by Zero) 에러'** 를 방지할 때 기가 막히게 쓰입니다

```sql
SELECT NULLIF('A', 'A');   -- 결과: NULL (두 값이 같으니까 폭파!)
SELECT NULLIF('A', 'B');   -- 결과: 'A'  (다르니까 첫 번째 값 반환)

-- 💡 실무 꿀팁: 분모가 0일 때 에러 안 나게 안전장치 걸기
-- (total_count가 0이면 NULL이 되고, 분모가 NULL이면 결과도 에러 없이 NULL 반환)
SELECT amount / NULLIF(total_count, 0) FROM sales;
```

### ④ Oracle 전용 3단 콤보: `NVL2`

- **문법:** `NVL2(검사할_컬럼, NULL이_아닐_때_값, NULL일_때_값)`
- **특징:** 컬럼이 빵꾸가 안 났을 때와 났을 때를 한 번에 분기 처리하는 **Oracle 전용** 함수입니다. (IF-ELSE 로직과 동일)

```sql
-- 🔴 Oracle (보너스가 있으면 '보너스 대상자', NULL이면 '제외')
SELECT NVL2(bonus, '대상자', '제외') FROM employees;
```

---
## SQLD 빈출 함정 & 실무 에러

### ① `COALESCE` 데이터 타입 불일치 에러 

실무에서 가장 많이 내는 에러입니다. 대체할 값은 **반드시 기존 컬럼과 데이터 타입(Type)이 같아야 합니다.**

```sql
-- ❌ 나쁜 예: age는 숫자(INT)인데 문자열('비밀')을 넣으려고 함 -> 타입 에러 발생!
SELECT COALESCE(age, '비밀') FROM users;

-- ⭕️ 좋은 예: 둘 다 문자열로 캐스팅(형변환) 해주어야 함!
SELECT COALESCE(CAST(age AS VARCHAR), '비밀') FROM users;
-- Postgres라면? COALESCE(age::VARCHAR, '비밀')
```


### ② `AVG()` 함수와 NULL의 환장할 콜라보

`AVG`는 평균을 낼 때 NULL인 행(Row)을 **투명 인간 취급하여 분모에서 빼버립니다.**

- 100점, 50점, NULL(미응시자) 3명의 점수가 있을 때:
    - `AVG(score)` 👉 `(100+50) / 2명 = 75점` (미응시자를 빼버려서 평균이 높아짐!)
    - `AVG(COALESCE(score, 0))` 👉 `(100+50+0) / 3명 = 50점` (정확한 전체 평균!)

### ③ `NVL` vs `ISNULL` vs `IS NULL` 헷갈림 주의!

- `NVL(컬럼, 0)`: Oracle에서 NULL을 0으로 **바꿔주는 '함수'**.
- `ISNULL(컬럼, 0)`: SQL Server에서 NULL을 0으로 **바꿔주는 '함수'**.
- `컬럼 IS NULL`: `WHERE` 절에서 이 컬럼이 NULL인지 **검색하는 '조건 연산자'**. (완전히 다른 역할입니다!)