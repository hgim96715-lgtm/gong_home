---
aliases:
  - 테이블생성
  - DDL
  - CREATE
  - DROP
  - 구조변경
  - TRUNCATE
  - ALTER
tags:
  - SQL
related:
  - "[[Python_Database_Connect]]"
  - "[[SQL_Data_Types]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DML_CRUD]]"
  - "[[SQL_DCL_Grant_Revoke]]"
---
# SQL DDL — 데이터베이스 뼈대 세우고 부수기

## 한 줄 요약

> **"DDL(Data Definition Language) 은 테이블 같은 구조물(뼈대)을 생성(`CREATE`), 변경(`ALTER`), 삭제(`DROP`·`TRUNCATE`) 하는 데이터 정의 명령어다."**

---

## 왜 필요한가?

건물을 지을 때 설계도 없이 시멘트부터 부을 수 없듯이, 데이터(DML)를 넣으려면 그 데이터를 담을 방(테이블)이 먼저 필요하다. "이 방에는 숫자만 들어올 수 있어!", "빈칸은 절대 안 돼!" 같은 규칙(제약조건)을 미리 세워두어야 나중에 쓰레기 데이터가 쌓이는 것을 막을 수 있다.

**실무 vs SQLD:**

- **PostgreSQL 실무** → 초기 스키마 설계·마이그레이션 때 주로 사용
- **SQLD 시험 (Oracle)** → 제약조건 종류, `ALTER` 문법, `TRUNCATE` vs `DELETE` 차이가 매번 출제

---

## ⚠️ Oracle DDL 의 암시적 COMMIT — 절대 잊으면 안 되는 함정

> **Oracle 에서 DDL 문장을 실행하면 내부적으로 트랜잭션을 종료시킨다.** 즉, DDL 실행 전에 미확정 상태로 있던 DML 이 있다면 **자동으로 COMMIT 되어버린다.**

```sql
-- Oracle 에서
INSERT INTO 사원 VALUES ('001', '홍길동');  -- 미확정 (COMMIT 안 함)
INSERT INTO 사원 VALUES ('002', '이순신');  -- 미확정 (COMMIT 안 함)

CREATE TABLE 임시 (id NUMBER);              -- DDL 실행 → 자동 COMMIT 발동!
-- ↑ 이 순간 위의 INSERT 2개도 함께 확정(영구 저장)되어버림

ROLLBACK;  -- 이미 늦었다. INSERT 는 되돌릴 수 없다!
```

|DB|DML 커밋 방식|DDL 커밋 방식|
|---|---|---|
|**Oracle**|수동 COMMIT 필요|**자동 COMMIT** (묵시적 실행) ⚠️|
|**PostgreSQL**|수동 COMMIT 필요|수동 COMMIT 필요|
|**SQL Server**|수동 COMMIT 필요|수동 COMMIT 필요|

> Oracle 에서 DDL 은 **트랜잭션을 종료시키는 폭탄**이다. DML 작업 중간에 CREATE · DROP · ALTER 를 치는 순간 앞서 쌓아두던 변경 사항이 전부 확정된다. → COMMIT / ROLLBACK 상세 → [[SQL_Database_Transactions_TCL]] 참고

---

---

# ① CREATE TABLE — 뼈대 세우기

## 이름 규칙 (Naming Rule)

- **시작은 무조건 알파벳(문자)으로!** (숫자·특수문자로 시작하면 즉시 에러)
- 허용 문자: `A-Z` `a-z` `0-9` `_` `$` `#`
- 실무 관행: 소문자 + 언더바 **스네이크 케이스** (예: `user_order_log`)
- 같은 스키마 내에서 테이블 이름 중복 불가
- `SELECT` `FROM` `TABLE` 같은 **예약어는 이름으로 쓸 수 없다**
- Oracle 이름 최대 길이: **30바이트**

## 기본 문법

```sql
CREATE TABLE 테이블명 (
    컬럼명1 데이터타입(크기),   -- 컬럼명 뒤에 데이터 타입과 크기 명시
    컬럼명2 데이터타입(크기),   -- 컬럼과 컬럼 사이는 콤마(,)로 구분
    컬럼명3 데이터타입(크기)    -- ⚠️ 마지막 컬럼에는 콤마 없음! 초보자 단골 실수
);                             -- 문장 끝은 반드시 세미콜론(;)으로 마무리
```

## 실전 예제

```sql
CREATE TABLE 사원 (
    사번     VARCHAR(10)  PRIMARY KEY,              -- UNIQUE + NOT NULL
    이름     VARCHAR(50)  NOT NULL,                 -- 빈칸 절대 금지
    부서코드  VARCHAR(10)  REFERENCES 부서(부서코드),  -- FK: 부서 테이블 참조
    입사일   DATE         DEFAULT CURRENT_DATE,     -- 값 없으면 오늘 날짜 자동 입력
    연봉     NUMERIC      CHECK (연봉 > 0)            -- 마이너스 연봉 차단
);
```

---

## 주요 데이터 타입 비교

|종류|SQLD / Oracle|PostgreSQL|설명|
|---|---|---|---|
|고정 문자|`CHAR(n)`|`CHAR(n)`|n자리 고정. 남는 공간은 공백으로 채움|
|가변 문자|`VARCHAR2(n)`|`VARCHAR(n)`|입력한 길이만큼만 저장|
|숫자|`NUMBER(p, s)`|`NUMERIC(p, s)`|p: 전체 자릿수, s: 소수점 자릿수|
|정수|`NUMBER(n)`|`INTEGER` / `BIGINT`|PostgreSQL 은 정수 전용 타입 제공|
|날짜|`DATE`|`DATE` / `TIMESTAMP`|PostgreSQL DATE 는 날짜만, TIMESTAMP 는 시간 포함|
|대용량 텍스트|`CLOB`|`TEXT`|PostgreSQL TEXT 는 크기 제한 없음|

> **SQLD 시험은 Oracle 기준** → `VARCHAR2`, `NUMBER` 사용법에 익숙해질 것 
> **PostgreSQL 실무** → `VARCHAR`, `INTEGER`, `TEXT`, `TIMESTAMP` 사용 빈도 높음

---

## NULL 과 DEFAULT

- **`NULL` / `NOT NULL`:** 빈칸 허용 여부. 명시하지 않으면 기본적으로 `NULL` 허용.
- **`DEFAULT`:** 데이터 입력 시 값을 안 넣으면 자동으로 채워줄 기본값.

> DATE 컬럼에 값을 넣을 때 Oracle 은 `TO_DATE()`, PostgreSQL 은 `::DATE` 로 명시적 변환을 권장한다. 
> 타입 변환·INSERT 상세 문법 → [[SQL_DML_CRUD]] · [[SQL_Type_Casting]] 참고

---

---

# ② 제약조건 (Constraints) 5가지

|제약조건|설명|특이사항|
|---|---|---|
|`PRIMARY KEY` (PK)|각 행을 유일하게 식별|`UNIQUE` + `NOT NULL` 결합. 테이블당 **1개만** 가능. **생성하지 않아도 된다** (필수 아님)|
|`FOREIGN KEY` (FK)|다른 테이블의 PK 를 참조|참조 무결성 유지|
|`UNIQUE` (UK)|중복값 허용 안 함|`NULL` 은 중복 허용 ⚠️|
|`CHECK`|조건에 맞는 데이터만 허용|예: `나이 > 0`, `성별 IN ('M', 'F')`|
|`NOT NULL`|빈칸 절대 금지|NULL 입력 시 즉시 에러|

> **💡 PK 는 필수가 아니다.** 테이블을 만들 때 PK 를 반드시 지정해야 하는 건 아니다. PK 없이도 테이블은 생성된다. 단, PK 가 없으면 행을 유일하게 식별할 방법이 없어서 중복 데이터가 쌓일 수 있고, 다른 테이블에서 FK 로 참조하기도 어려워진다. 실무에서는 거의 항상 PK 를 만드는 것이 원칙.

> **`UNIQUE` + `NOT NULL` 조합은 후보키(Candidate Key)가 된다.**

```
후보키 = 행을 유일하게 식별할 수 있는 최소한의 컬럼 집합
PRIMARY KEY = 후보키 중 대표로 선택된 것 1개
UNIQUE + NOT NULL = PK 로 선택되지 않았지만 후보키 역할을 할 수 있는 컬럼
```

```sql
CREATE TABLE 사원 (
    사번   VARCHAR(10) PRIMARY KEY,   -- 대표 후보키 → PK
    이메일 VARCHAR(100) UNIQUE NOT NULL -- UNIQUE + NOT NULL → 후보키 역할
    -- 이메일도 행을 유일하게 식별할 수 있으므로 후보키
);
```

> **⚠️ UNIQUE 만으로는 후보키가 안 된다.** NULL 이 여러 개 들어올 수 있어서 유일성이 보장되지 않기 때문이다.

---

## FK 참조 무결성 옵션

부모 테이블의 데이터가 **삭제·수정** 될 때 자식 테이블을 어떻게 처리할지 정하는 규칙이다.

```sql
ALTER TABLE 사원
ADD CONSTRAINT fk_부서
FOREIGN KEY (부서코드) REFERENCES 부서(부서코드)
ON DELETE CASCADE;  -- ← 여기에 옵션을 붙인다
```

| 옵션            | 동작                                           | 비유                      |
| ------------- | -------------------------------------------- | ----------------------- |
| `CASCADE`     | 부모 삭제 시 자식도 **같이 삭제**                        | 부서 삭제 → 소속 사원도 삭제       |
| `SET NULL`    | 부모 삭제 시 자식 FK 를 **NULL 로**                   | 부서 삭제 → 사원의 부서코드 = NULL |
| `SET DEFAULT` | 부모 삭제 시 자식 FK 를 **DEFAULT 값으로**              | 부서 삭제 → 사원을 '미배정팀' 으로   |
| `RESTRICT`    | 자식이 참조 중이면 부모 삭제 **즉시 거부**                   | 사원 있으면 부서 삭제 불가         |
| `NO ACTION`   | `RESTRICT` 와 동일하나 **트랜잭션 종료 시점** 에 체크        | SQLD 시험에서 개념 구분 주의!     |
| `DEPENDENT`   | 부모 테이블에 **PK 가 없는 경우** 자식 데이터 입력 자체를 허용하지 않음 | PK 없는 부모 → 자식 INSERT 차단 |

> **RESTRICT vs NO ACTION** `RESTRICT` 는 SQL 실행 **즉시** 에러, `NO ACTION` 은 트랜잭션 **종료 시점** 에 에러. PostgreSQL 에서는 사실상 동일하게 동작하지만, SQLD 시험에서 개념 구분이 나올 수 있다.

> **DEPENDENT 추가 설명** 일반적인 FK 는 "부모 PK 에 존재하는 값만 자식에 넣을 수 있다" 는 규칙이다. DEPENDENT 는 한 발 더 나아가 "부모 테이블 자체에 PK 가 없으면 자식 INSERT 를 아예 막는다" 는 옵션이다. SQLD 시험 개념 문제로 종종 출제된다.

---
## CASCADE vs CASCADE CONSTRAINTS — 헷갈리는 이름, 완전히 다른 개념

| **구분**        | **CASCADE (FK 옵션)**                      | **CASCADE CONSTRAINTS (DDL 옵션)**                   |
| ------------- | ---------------------------------------- | -------------------------------------------------- |
| **사용 위치**     | `CREATE TABLE` 시 FK 정의 부분                | `DROP TABLE` 또는 `ALTER TABLE` 문                    |
| **대상**        | **데이터 (행)**                              | **제약 조건 (구조)**                                     |
| **주요 동작**     | 부모 테이블의 행 삭제 시, 이를 참조하는 자식 테이블의 행도 함께 삭제 | 테이블 삭제 시, 해당 테이블을 참조하고 있는 모든 외래 키(FK) 제약 조건을 먼저 삭제 |
| **자식 데이터 상태** | 함께 삭제됨                                   | 그대로 유지됨 (제약 조건만 사라짐)                               |
| **주요 목적**     | 참조 무결성 유지를 위한 자동 삭제                      | 제약 조건으로 인해 테이블 삭제가 안 되는 상황 해결                      |

```sql
-- CASCADE (FK 옵션): 데이터(행) 를 지울 때의 동작 규칙
FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
ON DELETE CASCADE;
-- departments 의 행 삭제 → employees 의 해당 행도 같이 삭제

-- CASCADE CONSTRAINTS (DDL 옵션): 테이블 구조를 날릴 때 사용
DROP TABLE departments CASCADE CONSTRAINTS;  -- Oracle
DROP TABLE departments CASCADE;              -- PostgreSQL
-- departments 테이블 삭제
-- + employees 의 FK 제약조건도 자동 삭제
-- (employees 테이블과 데이터는 그대로 살아있음)
```

```text
CASCADE CONSTRAINTS 없이 DROP TABLE departments
→ ❌ 에러: employees 가 참조 중이라 삭제 불가

CASCADE CONSTRAINTS 붙이고 DROP TABLE departments
→ ✅ departments 테이블 삭제
   + employees 의 FK 제약조건(구조)도 삭제
   (employees 의 데이터는 그대로)
```

---
---

# ③ CTAS — 기존 테이블 복사해서 만들기

`CREATE TABLE ... AS SELECT ...` 의 줄임말. 실무에서는 **'씨타스'** 라고 부른다.

```sql
-- 전체 복사
CREATE TABLE 사원_백업 AS SELECT * FROM 사원;

-- 조건부 복사 (실적 90 이상만)
CREATE TABLE 우수사원 AS
SELECT 사번, 이름, 연봉 FROM 사원 WHERE 실적 > 90;

-- ⭐ 뼈대만 복사 (구조만, 데이터 없이) ← SQLD 단골 기출!
CREATE TABLE 사원_빈껍데기 AS
SELECT * FROM 사원 WHERE 1 = 2;
-- 1 = 2 는 절대 참이 될 수 없는 조건 → 데이터 0건 + 컬럼 구조만 복사
```

## ⚠️ CTAS 의 치명적 함정 — SQLD 킬러 문항

|항목|복사 여부|
|---|:-:|
|컬럼명|✅|
|데이터 타입|✅|
|데이터(값)|✅|
|`NOT NULL`|⚠️ DB 에 따라 다름|
|`PRIMARY KEY`|❌|
|`FOREIGN KEY`|❌|
|`CHECK`|❌|
|`DEFAULT`|❌|

> **CTAS 후 PK 가 필요하다면 반드시 `ALTER TABLE` 로 수동으로 다시 달아줘야 한다.**

---

---

# ④ ALTER TABLE — 뼈대 뜯어고치기

이미 만든 테이블에 컬럼을 추가하거나, 이름·타입을 바꾸거나, 제약조건을 추가·삭제하는 명령이다.

## 전체 문법 한눈에 비교

| 작업           | Oracle (SQLD)               | PostgreSQL (실무)                |
| ------------ | --------------------------- | ------------------------------ |
| 컬럼 추가        | `ADD (컬럼명 타입)`              | `ADD COLUMN 컬럼명 타입`            |
| 컬럼 삭제        | `DROP COLUMN 컬럼명`           | `DROP COLUMN 컬럼명`              |
| 컬럼 이름 변경     | `RENAME COLUMN 기존 TO 새이름`   | `RENAME COLUMN 기존 TO 새이름`      |
| **컬럼 속성 변경** | **`MODIFY (컬럼명 타입)`**       | **`ALTER COLUMN 컬럼명 TYPE 타입`** |
| 제약조건 추가      | `ADD CONSTRAINT 이름 종류 (컬럼)` | `ADD CONSTRAINT 이름 종류 (컬럼)`    |
| 제약조건 삭제      | `DROP CONSTRAINT 이름`        | `DROP CONSTRAINT 이름`           |


---

## A. 컬럼 추가 · 삭제 · 이름 변경

```sql
-- 컬럼 추가
-- 🔴 Oracle: COLUMN 키워드 없음, 괄호 사용 가능
ALTER TABLE 사원 ADD 이메일 VARCHAR2(100);
ALTER TABLE 사원 ADD (이메일 VARCHAR2(100));        -- 괄호 버전도 OK
ALTER TABLE 사원 ADD (이메일 VARCHAR2(100), 전화번호 VARCHAR2(20)); -- 여러 컬럼 동시 추가

-- 🐘 PostgreSQL: ADD COLUMN (COLUMN 키워드 필수)
ALTER TABLE 사원 ADD COLUMN 이메일 VARCHAR(100);

-- 컬럼 삭제 🚨 복구 불가! (Oracle / PostgreSQL 공통)
ALTER TABLE 사원 DROP COLUMN 이메일;

-- 컬럼 이름 변경 (Oracle / PostgreSQL 공통)
ALTER TABLE 사원 RENAME COLUMN 이름 TO 성명;
```

>**Oracle 핵심:** `ADD` 뒤에 `COLUMN` 키워드를 쓰지 않는다. 괄호로 여러 컬럼을 한 번에 추가할 수 있다.
> **PostgreSQL 핵심:** `ADD COLUMN` 처럼 `COLUMN` 을 반드시 써야 한다.

---

## B. 컬럼 속성 변경 — ⚠️ DB 별 문법 다름!

```sql
-- 🔴 Oracle (SQLD): MODIFY
ALTER TABLE 사원
MODIFY (이름 VARCHAR2(100));

-- 여러 컬럼 동시에 변경할 때도 MODIFY
ALTER TABLE 사원
MODIFY (이름 VARCHAR2(100), 연봉 NUMBER(10));

-- 🐘 PostgreSQL (실무): ALTER COLUMN ... TYPE
ALTER TABLE 사원
ALTER COLUMN 이름 TYPE VARCHAR(100);
```

> **Oracle 은 `MODIFY`, PostgreSQL 은 `ALTER COLUMN ... TYPE`** 이 차이가 SQLD 시험에서 가장 많이 나오는 함정이다. 
> 절대 혼용하지 말 것.

---
## B-1. ALTER 로 DEFAULT 지정 시 — 적용 시점 주의 ⭐️

> **ALTER TABLE 로 DEFAULT 를 추가하면 기존 데이터에는 소급 적용되지 않는다.** 
> **ALTER 명령 실행 이후에 INSERT 되는 데이터부터만 DEFAULT 가 적용된다.**

```sql
-- DEFAULT 가 없던 컬럼에 ALTER 로 DEFAULT 추가
ALTER TABLE 사원
MODIFY (입사일 DATE DEFAULT CURRENT_DATE);             -- 🔴 Oracle

-- ALTER TABLE 사원 ALTER COLUMN 입사일 SET DEFAULT CURRENT_DATE;  -- 🐘 PostgreSQL
```

```text
ALTER 실행 전 기존 데이터 (이미 쌓여있던 행):
┌─────┬────────┬──────────┐
│ 사번  │ 이름   │ 입사일   │
├─────┼────────┼──────────┤
│ 001  │ 홍길동 │ NULL     │  ← ALTER 해도 소급 적용 안 됨! 그대로 NULL 유지
│ 002  │ 이순신 │ NULL     │  ← 마찬가지로 NULL 유지
└─────┴────────┴──────────┘

ALTER 실행 후 새로 INSERT:
INSERT INTO 사원(사번, 이름) VALUES ('003', '강감찬');
-- 입사일 값을 안 넣었지만 → DEFAULT 적용 → 오늘 날짜 자동 입력 ✅

결과:
│ 003  │ 강감찬 │ 2026-03-04 │  ← 이후 데이터부터만 DEFAULT 적용
```

### 기존 NULL 데이터에도 값을 채우고 싶다면 — UPDATE 따로 실행

```sql
-- ALTER 로는 안 됨. UPDATE 로 직접 채워야 한다.
UPDATE 사원
SET 입사일 = CURRENT_DATE
WHERE 입사일 IS NULL;
```

### DEFAULT 삭제

```sql
-- 🔴 Oracle: DEFAULT 를 NULL 로 덮어씌워 제거
ALTER TABLE 사원 MODIFY (입사일 DATE DEFAULT NULL);

-- 🐘 PostgreSQL: DROP DEFAULT 명시
ALTER TABLE 사원 ALTER COLUMN 입사일 DROP DEFAULT;
```

>❌ **SQLD 시험 오답 유형:** "ALTER TABLE 로 DEFAULT 를 추가하면 기존 행의 값도 변경된다" 
>→ **틀림!** 기존 데이터는 변경되지 않고, **이후 INSERT 부터만** 적용된다.

---

## C. 제약조건 추가 — ADD CONSTRAINT ⭐️

```
ALTER TABLE 테이블명
ADD CONSTRAINT  [이름]  [종류]  ([컬럼]);
                 ↑
                 이름을 빠뜨리기 쉽다!
```

> **제약조건 이름 작명 규칙:** `pk_테이블명` / `fk_참조대상` / `uq_컬럼명` / `ck_컬럼명`

```sql
-- ✅ PK 추가 (이름 필수!)
ALTER TABLE 사원
ADD CONSTRAINT pk_사원 PRIMARY KEY (사번);
--              ↑        ↑           ↑
--           제약조건명   PK 키워드    PK 컬럼

-- ❌ 자주 하는 실수: 이름 생략
ALTER TABLE 사원
ADD PRIMARY KEY (사번);  -- Oracle 에러, PostgreSQL 은 자동 이름 부여

-- FK 추가
ALTER TABLE 사원
ADD CONSTRAINT fk_부서 FOREIGN KEY (부서코드) REFERENCES 부서(부서코드);
--              ↑        ↑              ↑              ↑
--         제약조건명    FK 키워드         내 컬럼        상대테이블.상대컬럼
```

---

## D. 제약조건 삭제

```sql
-- ADD CONSTRAINT 때 지어둔 이름으로 삭제
ALTER TABLE 사원 DROP CONSTRAINT pk_사원;
ALTER TABLE 사원 DROP CONSTRAINT fk_부서;
```

> ADD 할 때 이름을 지어두는 이유: DROP 할 때 이 이름으로 지목하기 위해서다. 이름이 없으면 DB 내부 자동 생성 이름을 찾아야 해서 번거롭다.

> ALTER TABLE 상세 → [[SQL_DDL_Create#④ ALTER TABLE — 뼈대 뜯어고치기|ALTER TABLE]] 참고

---

---

# ⑤ DROP · TRUNCATE · DELETE 비교 — SQLD 단골 출제

## DELETE vs TRUNCATE vs DROP

|구분|`DELETE`|`TRUNCATE`|`DROP`|
|---|:-:|:-:|:-:|
|데이터|조건부 삭제|전체 삭제|전체 삭제|
|구조(뼈대)|✅ 유지|✅ 유지|❌ 삭제|
|롤백|✅ 가능|❌ 불가|❌ 불가|
|속도|🐢 느림|⚡ 빠름|⚡ 빠름|
|WHERE 조건|✅ 가능|❌ 불가|❌ 불가|
|분류|DML|DDL|DDL|
|로그 기록|✅ 행 단위 기록|❌ 기록 안 함|❌ 기록 안 함|

```
비유로 기억하기:
DELETE   → 🧹 물건을 하나씩 골라서 치운다 (느리지만 롤백 가능)
TRUNCATE → 🪣 방 안을 전부 쏟아버린다 (빠르지만 롤백 불가, 방은 남음)
DROP     → 💣 건물 자체를 폭파한다 (복구 불가, 방도 없어짐)
```

## WHERE 없는 DELETE 는 TRUNCATE 와 같은가? ⭐️

```sql
DELETE FROM 사원;          -- WHERE 없이 전체 삭제
TRUNCATE TABLE 사원;       -- 전체 초기화
```

**결과는 같아 보이지만 동작 방식이 완전히 다르다.**

|비교 항목|`DELETE` (WHERE 없음)|`TRUNCATE`|
|---|---|---|
|최종 결과|모든 행 삭제|모든 행 삭제|
|롤백|✅ 가능 (트랜잭션 내)|❌ 불가|
|속도|🐢 느림 (행마다 로그 기록)|⚡ 빠름 (로그 없이 일괄 삭제)|
|트리거|✅ 행 단위 트리거 실행|❌ 트리거 실행 안 됨|
|AUTO_INCREMENT 초기화|❌ 번호 유지|✅ 번호 1 부터 다시 시작|

> **결론:** 롤백이 필요하거나 트리거가 걸려있다면 `DELETE`, 빠르게 전체 비우고 싶다면 `TRUNCATE` 를 써야 한다. WHERE 없는 DELETE 를 TRUNCATE 대용으로 쓰면 **속도 차이**가 크게 난다.

---

## 테이블 삭제 (DROP TABLE)

```sql
-- 기본
DROP TABLE 테이블명;

-- CASCADE: FK 로 엮인 제약조건까지 강제로 끊고 삭제
DROP TABLE 테이블명 CASCADE CONSTRAINTS;  -- Oracle
DROP TABLE 테이블명 CASCADE;              -- PostgreSQL

-- RESTRICT: FK 참조가 있으면 삭제 거부 (안전 장치)
DROP TABLE 테이블명 RESTRICT;             -- PostgreSQL / MySQL
```

|상황|`DROP`|`DROP CASCADE`|`DROP RESTRICT`|
|---|:-:|:-:|:-:|
|FK 참조 없을 때|✅ 정상 삭제|✅ 정상 삭제|✅ 정상 삭제|
|FK 참조 있을 때|❌ 에러|✅ FK 끊고 강제 삭제|❌ 명시적 에러|

> **CASCADE** → FK 제약조건만 끊는 것. 자식 테이블 자체는 살아있음. **RESTRICT** → 참조 있으면 명시적 거부. 운영 환경 안전장치로 활용.

---

## 테이블 이름 변경 (RENAME)

```sql
-- 🔴 Oracle (SQLD)
RENAME 기존테이블명 TO 새테이블명;
RENAME 사원 TO 직원;

-- 🐘 PostgreSQL (실무)
ALTER TABLE 기존테이블명 RENAME TO 새테이블명;
ALTER TABLE 사원 RENAME TO 직원;
```

---

---

# Oracle vs PostgreSQL 문법 차이 한눈에 보기

|작업|Oracle (SQLD)|PostgreSQL (실무)|
|---|:-:|:-:|
|가변 문자 타입|`VARCHAR2(n)`|`VARCHAR(n)`|
|숫자 타입|`NUMBER(p,s)`|`NUMERIC(p,s)`|
|날짜 리터럴|`TO_DATE('2021-12-31', 'YYYY-MM-DD')`|`'2021-12-31'::DATE`|
|타입 자동 변환|관대함 (숫자→문자 자동)|엄격함 (명시 권장)|
|컬럼 속성 변경|`MODIFY`|`ALTER COLUMN … TYPE`|
|테이블 이름 변경|`RENAME A TO B`|`ALTER TABLE A RENAME TO B`|
|DROP + FK 강제 삭제|`CASCADE CONSTRAINTS`|`CASCADE`|

---
