---
aliases:
  - 형변환
  - TO_CHAR
  - TO_DATE
  - 명시적 형변환
  - 암시적 형변환
tags:
  - SQL
related:
  - "[[SQL_Data_Types]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_String_Functions]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_Numeric_Functions]]"
---
# 명시적/암시적 형변환

##  개념 한 줄 요약 (Concept Summary)

**"숫자를 문자로, 문자를 날짜로! 데이터의 옷(타입)을 상황에 맞게 갈아입히는 마법."**
문자열 `'123'`과 숫자 `123`은 눈으로 보기엔 같지만 DB 입장에선 완전히 다른 종족입니다. 
이 둘을 연산하거나 비교하기 위해 타입을 통일시켜 주는 작업이 바로 형변환(Casting)입니다.

---
## 형변환의 2가지 방식: 눈치껏 vs 대놓고

### ① 암시적 형변환 (Implicit Casting) = "DB가 눈치껏"

사용자가 명령하지 않아도, DB 엔진이 에러를 뱉기 전에 **"네가 이걸 원한 거지?" 하면서 몰래 타입을 바꿔서** 처리해 주는 방식입니다.

**예시:** `SELECT '100' + 200;`

- 문자열 `'100'`을 DB가 알아서 숫자 `100`으로 바꾼 뒤 더해서 `300`을 반환합니다. (MySQL, SQL Server 기준. _참고: Oracle은 더해지지만, Postgres는 에러를 뱉을 정도로 엄격합니다._)

**🚨 치명적 단점 (실무/시험 공통):** 
- 성능 저하의 주범입니다! DB가 몰래 타입을 변환하느라 인덱스(Index)를 타지 못하고 테이블 전체를 뒤지는 **풀 테이블 스캔(Full Table Scan)** 이 발생할 수 있습니다.

### ② 명시적 형변환 (Explicit Casting) = "대놓고 멱살 잡기"

사용자가 `CAST`나 `TO_CHAR` 같은 변환 함수를 써서 **"이 컬럼은 무조건 이 타입으로 바꿔!"** 라고 명확하게 지시하는 방식입니다.

- **장점:** 쿼리 읽기가 편해지고, DB의 오작동(성능 저하)을 완벽하게 막아줍니다. **실무에서는 100% 명시적 형변환을 권장합니다.**

---
## 명시적 형변환 핵심 함수

### (표준 SQL): `CAST()`

어떤 DB를 쓰든 다 알아듣는 가장 표준적인 형변환 함수입니다.

- **문법:** `CAST(대상 AS 바꿀_타입)`

```sql
SELECT CAST('2026-02-24' AS DATE);  -- 문자를 날짜로!
SELECT CAST(123.45 AS INT);         -- 실수를 정수로! (123 반환)
SELECT CAST(123 AS VARCHAR);        -- 숫자를 문자로!
```

### PostgreSQL 특화: `::` (초간편 캐스팅)

실무에서 Postgres를 쓸 때 가장 사랑받는 문법입니다. `CAST()`를 치기 귀찮을 때 뒤에 `::`만 붙이면 끝납니다.

```sql
SELECT '2026-02-24'::DATE;     -- 문자를 날짜로!
SELECT 123.45::INT;            -- 실수를 정수로!
SELECT 123::VARCHAR;           -- 숫자를 문자로!
```

### Oracle 특화 (SQLD 초핵심 3대장!): `TO_` 패밀리

Oracle은 타입을 바꿀 때 `TO_`로 시작하는 전용 함수를 사용합니다.

- **`TO_CHAR(숫자/날짜, '포맷')`**: 무언가를 **문자(String)** 로 바꿉니다. (날짜 예쁘게 포장할 때 많이 썼죠?)
- **`TO_NUMBER('문자열')`**: 글자로 된 숫자를 진짜 **숫자(Number)** 로 바꿉니다.
- **`TO_DATE('문자열', '포맷')`**: 글자로 된 날짜를 진짜 **날짜(Date)** 로 바꿉니다.

```sql
-- 🔴 Oracle (SQLD 단골 출제)
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD') FROM DUAL;     -- 날짜 -> 문자
SELECT TO_NUMBER('12345') + 100 FROM DUAL;           -- 문자 -> 숫자 (12445)
SELECT TO_DATE('20260224', 'YYYYMMDD') FROM DUAL;    -- 문자 -> 날짜
```

###  SQL Server (MSSQL) 특화: `CONVERT()`

MSSQL은 `CAST()`도 지원하지만, 날짜 포맷팅 등 디테일한 변환을 할 때는 `CONVERT()`를 밥 먹듯이 씁니다.

- **문법:** `CONVERT(바꿀_타입, 대상, [스타일코드])`

```sql
SELECT CONVERT(INT, '123');              -- 문자를 정수로!
SELECT CONVERT(VARCHAR, GETDATE(), 23);  -- 날짜를 문자로! (스타일 23 = YYYY-MM-DD)
```

---
##  초보자가 자주 하는 실수 & 실무 팁 (Common Mistakes)

### ① "인덱스(Index) 박살 내기" (가장 흔한 실무 장애)

`user_id` 컬럼이 문자열(`VARCHAR`) 타입으로 저장되어 있다고 가정해 봅시다.

```sql
-- ❌ 나쁜 예 (암시적 형변환 발생)
SELECT * FROM users WHERE user_id = 12345;
```

**문제점:** 
컬럼은 문자인데, 검색조건에 숫자(`12345`)를 넣었습니다. DB는 비교를 위해 테이블에 있는 수백만 건의 `user_id`를 전부 숫자로 임시 변환(암시적 형변환)하느라 인덱스를 타지 못하고 서버가 뻗어버릴 수 있습니다!

```sql
-- ⭕️ 좋은 예 (타입을 똑같이 맞춰줌)
SELECT * FROM users WHERE user_id = '12345';
```

### ② "날짜 포맷 안 맞추고 그냥 밀어 넣기"

문자열을 날짜로 바꿀 때, 포맷(형식)을 안 알려주면 DB가 당황합니다.

```sql
-- 🔴 Oracle 기준 에러 위험
SELECT TO_DATE('2026/02/24') FROM DUAL; 

-- ⭕️ 올바른 방법 (DB에게 읽는 방법을 알려주기)
SELECT TO_DATE('2026/02/24', 'YYYY/MM/DD') FROM DUAL;
```
