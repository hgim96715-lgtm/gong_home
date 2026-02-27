---
aliases:
  - 테이블생성
  - DDL
  - CREATE
  - DROP
  - 구조변경
tags:
  - Python
related:
  - "[[Python_Database_Connect]]"
  - "[[SQL_Data_Types]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DML_CRUD]]"
---
# DDL : 데이터베이스 뼈대 세우고 부수기 (CREATE, ALTER, DROP)

## 한줄 요약

DDL(Data Definition Language)은 데이터베이스에 테이블(표)이나 인덱스 같은 구조물(뼈대)을 처음 **생성(`CREATE`)** 하고, 구조를 **변경(`ALTER`)** 하거나, 아예 **삭제(`DROP`, `TRUNCATE`)** 하는 데이터 정의 명령어다.

---
## Why is it needed(왜 필요한가)

건물을 지을 때 설계도 없이 무작정 시멘트부터 부을 수 없듯이, 데이터(DML)를 넣으려면 그 데이터를 담을 튼튼하고 규칙적인 방(테이블)이 먼저 필요하다. 이때 "이 방에는 숫자만 들어올 수 있어!", "빈칸은 절대 안 돼!" 같은 엄격한 규칙(제약조건)을 미리 세워두어야 나중에 쓰레기 데이터가 쌓이는 것을 막을 수 있다.

---
## Practical Context(실무맥락)

실무 파이프라인(PostgreSQL)에서는 초기 스키마 설계나 마이그레이션 때 주로 쓰며, **SQLD 시험(Oracle)** 에서는 제약조건의 종류와 `ALTER` 문법의 세세한 차이, 그리고 `TRUNCATE`와 `DELETE`의 차이를 묻는 문제가 매번 출제된다. 특히 **PostgreSQL과 Oracle의 문법이 살짝 다른 부분**이 있어서 이 차이를 정확히 짚고 넘어가는 게 핵심이다!

---
## 테이블 생성 시 반드시 지켜야 할 절대 규칙

### 이름 규칙 (Naming Rule)

- **시작은 무조건 알파벳(문자)으로!** (숫자나 특수문자로 시작하면 바로 에러)
- 이름은 `A-Z`, `a-z`, `0-9`와 특수문자 `_`, `$`, `#`만 허용된다. 
	- 실무에서는 소문자 + 언더바 조합인 **스네이크 케이스**를 가장 많이 쓴다. (예: `user_order_log`)
- 다른 테이블 이름과 **중복될 수 없다.** (하나의 스키마/계정 내에서)
- 예약어(`SELECT`, `FROM`, `TABLE` 등)는 이름으로 쓸 수 없다.
- (오라클 기준) 이름의 길이는 최대 30바이트까지만 가능하다.

### 문법 규칙 (Syntax Rule)

```sql
CREATE TABLE 테이블명 (
    -- 컬럼 정의는 반드시 괄호 () 안에 기술

    컬럼명1 데이터타입(크기),   -- 컬럼명 뒤에 데이터 타입과 크기 명시
    컬럼명2 데이터타입(크기),   -- 컬럼과 컬럼 사이는 콤마(,)로 구분
    컬럼명3 데이터타입(크기)    -- 마지막 컬럼에는 콤마 없음!
);                         -- 문장 끝은 반드시 세미콜론(;)으로 마무리
```

- 컬럼들의 정의는 **괄호 `()` 안에** 기술한다.
- 각 컬럼은 **콤마 `,`** 로 구분한다.
- **마지막 컬럼 뒤에는 콤마를 붙이지 않는다.** (초보자 단골 실수 ⚠️)
- 문장의 끝은 반드시 **세미콜론 `;`** 으로 마무리한다.
- 컬럼명 뒤에는 반드시 **데이터 타입과 크기**를 명시한다.

### 주요 데이터 타입 비교 (SQLD vs PostgreSQL)

| 종류      | SQLD / Oracle  | PostgreSQL           | 설명                                     |
| ------- | -------------- | -------------------- | -------------------------------------- |
| 고정 문자   | `CHAR(n)`      | `CHAR(n)`            | n자리 고정 길이, 남는 공간은 공백으로 채움              |
| 가변 문자   | `VARCHAR2(n)`  | `VARCHAR(n)`         | 입력한 길이만큼만 저장                           |
| 숫자      | `NUMBER(p, s)` | `NUMERIC(p, s)`      | p: 전체 자릿수, s: 소수점 자릿수                  |
| 정수      | `NUMBER(n)`    | `INTEGER` / `BIGINT` | PostgreSQL은 정수 전용 타입 제공                |
| 날짜      | `DATE`         | `DATE` / `TIMESTAMP` | PostgreSQL DATE는 날짜만, TIMESTAMP는 시간 포함 |
| 대용량 텍스트 | `CLOB`         | `TEXT`               | PostgreSQL TEXT는 크기 제한 없음              |

> [!tip] SQLD vs PostgreSQL 핵심 정리
> **SQLD 시험은 Oracle 기준**으로 출제된다.  
>→ `VARCHAR2`, `NUMBER` 사용법에 익숙해질 것
>
>**PostgreSQL 실무**에서는 
>→ `VARCHAR`, `INTEGER`, `TEXT`, `TIMESTAMP` 사용 빈도가 높다
>  📎 [[SQL_Data_Types|데이터 타입 상세 보기]]

---
## Code Core Points : `CREATE TABLE` (뼈대 세우기)

테이블을 만들 때는 컬럼명, 데이터 타입, 빈칸 허용 여부, 기본값, 제약조건을 명시한다.

### 기본 문법

```sql
CREATE TABLE 테이블명 (
    컬럼명1 데이터타입 [DEFAULT 기본값] [제약조건],
    컬럼명2 데이터타입 [제약조건],
    ...
);
```

### 실전 예제

```sql
CREATE TABLE 사원 (
    사번     VARCHAR(10)  PRIMARY KEY,            -- UNIQUE + NOT NULL
    이름     VARCHAR(50)  NOT NULL,               -- 빈칸 절대 금지
    부서코드  VARCHAR(10)  REFERENCES 부서(부서코드), -- FK: 부서 테이블 참조
    입사일   DATE         DEFAULT CURRENT_DATE,   -- 값 없으면 오늘 날짜 자동 입력
    연봉     NUMERIC      CHECK (연봉 > 0)          -- 마이너스 연봉 차단
);
```

---

### 필수 개념: `NULL` 과 `DEFAULT`

- **`NULL` / `NOT NULL`**: 빈칸 허용 여부. 명시하지 않으면 기본적으로 `NULL` 허용.
- **`DEFAULT`**: `INSERT` 시 값을 안 넣으면 자동으로 채워줄 기본값.

---

### 필수 개념: 제약조건 (Constraints) 5가지

| 제약조건 | 설명 | 특이사항 |
|---------|------|---------|
| `PRIMARY KEY` (PK) | 각 행을 유일하게 식별 | `UNIQUE` + `NOT NULL` 결합. 테이블당 **1개만** 가능 |
| `FOREIGN KEY` (FK) | 다른 테이블의 PK를 참조 | 참조 무결성 유지 |
| `UNIQUE` (UK) | 중복값 허용 안 함 | `NULL`은 중복 허용 ⚠️ |
| `CHECK` | 조건에 맞는 데이터만 허용 | 예: `나이 > 0`, `성별 IN ('M', 'F')` |
| `NOT NULL` | 빈칸 절대 금지 | NULL 입력 시 즉시 에러 |

---

### FK 참조 무결성 옵션

부모 테이블의 데이터가 **삭제/수정** 될 때, 자식 테이블을 어떻게 처리할지 정하는 규칙이다.

```sql
ALTER TABLE 사원
ADD CONSTRAINT fk_부서
FOREIGN KEY (부서코드) REFERENCES 부서(부서코드)
ON DELETE CASCADE;  -- ← 여기에 옵션을 붙인다
```

| 옵션 | 동작 | 비유 |
|------|------|------|
| `CASCADE` | 부모 삭제 시 자식도 **같이 삭제** | 부서 삭제 → 소속 사원도 삭제 |
| `SET NULL` | 부모 삭제 시 자식 FK를 **NULL로** | 부서 삭제 → 사원 부서코드 = NULL |
| `SET DEFAULT` | 부모 삭제 시 자식 FK를 **DEFAULT 값으로** | 부서 삭제 → 사원을 '미배정팀'으로 |
| `RESTRICT` | 자식이 참조 중이면 부모 **삭제 거부** (즉시 체크) | 사원 있으면 부서 삭제 불가 |
| `NO ACTION` | `RESTRICT`와 동일하나 **트랜잭션 종료 시점**에 체크 | SQLD 시험에서 개념 구분 주의! |

```sql
ON DELETE CASCADE      -- 부모 삭제 시 자식도 같이 삭제
ON DELETE SET NULL     -- 부모 삭제 시 자식 FK → NULL
ON DELETE SET DEFAULT  -- 부모 삭제 시 자식 FK → DEFAULT 값
ON DELETE RESTRICT     -- 자식 있으면 부모 삭제 즉시 거부
ON DELETE NO ACTION    -- 자식 있으면 트랜잭션 종료 시 거부
```

> 💡 **RESTRICT vs NO ACTION**
> `RESTRICT`는 SQL 실행 **즉시** 에러, `NO ACTION`은 트랜잭션 **종료 시점**에 에러.
> PostgreSQL에서는 사실상 동일하게 동작하지만, SQLD 시험에서 개념 구분이 나올 수 있다.

---
### CTAS — 기존 테이블 복사해서 만들기

`CREATE TABLE ... AS SELECT ...` 의 줄임말. 실무에선 **'씨타스'** 라고 부른다.

```sql
-- 기본 문법
CREATE TABLE 새테이블명 AS
SELECT 컬럼명 FROM 기존테이블명 WHERE 조건;
```
```sql
-- 1. 전체 복사
CREATE TABLE 사원_백업 AS
SELECT * FROM 사원;

-- 2. 조건부 복사 (실적 90 이상만)
CREATE TABLE 우수사원 AS
SELECT 사번, 이름, 연봉 FROM 사원 WHERE 실적 > 90;

-- 3. 뼈대만 복사 (구조만, 데이터 없이)  SQLD 단골 기출
CREATE TABLE 사원_빈껍데기 AS
SELECT * FROM 사원 WHERE 1 = 2;
-- 1 = 2는 절대 참이 될 수 없는 조건 → 데이터 0건 + 컬럼 구조만 복사
```

#### ⚠️ CTAS의 치명적 함정 — SQLD 킬러 문항 

CTAS로 복사되는 것과 안 되는 것을 반드시 구분해야 한다.

| 항목 | 복사 여부 |
|------|---------|
| 컬럼명 | ✅ 복사됨 |
| 데이터 타입 | ✅ 복사됨 |
| 데이터(값) | ✅ 복사됨 |
| `NOT NULL` | ⚠️ DB에 따라 다름 |
| `PRIMARY KEY` | ❌ 복사 안 됨 |
| `FOREIGN KEY` | ❌ 복사 안 됨 |
| `CHECK` | ❌ 복사 안 됨 |
| `DEFAULT` | ❌ 복사 안 됨 |

> CTAS 후 PK가 필요하다면 반드시 `ALTER TABLE`로 수동으로 다시 달아줘야 한다.

---
## Code Core Points : `ALTER TABLE` (뼈대 뜯어고치기)

이미 지어진 건물에 방을 추가하거나, 창문을 없애거나, 용도를 변경하는 작업이다. 
여기서 **SQLD(Oracle)와 실무(PostgreSQL)의 문법이 확연히 갈리니** 주의해야 한다!

### 컬럼 추가 (`ADD COLUMN`)

- 테이블의 맨 마지막에 새로운 컬럼이 추가된다.

```sql
-- Oracle / PostgreSQL 공통 
ALTER TABLE 테이블명 ADD 컬럼명 데이터타입;

-- 예제
ALTER TABLE 사원 ADD 이메일 VARCHAR(100);
```


###  컬럼 삭제 (`DROP COLUMN`)

- 안에 데이터가 꽉 차 있어도 얄짤없이 컬럼을 통째로 날려버린다. (복구 불가!)

```sql
-- Oracle / PostgreSQL 공통 
ALTER TABLE 테이블명 DROP COLUMN 컬럼명;

-- 예제 
ALTER TABLE 사원 DROP COLUMN 이메일;
```

### 컬럼 이름 변경 (`RENAME COLUMN`)

```sql
-- Oracle / PostgreSQL 공통
ALTER TABLE 테이블명 RENAME COLUMN 기존이름 TO 새이름;

-- 예제
ALTER TABLE 사원 RENAME COLUMN 이름 TO 성명;
```

###  제약조건 추가 (`ADD CONSTRAINT`)

- 테이블을 만든 후에 뒤늦게 PK나 FK를 걸고 싶을 때 쓴다.

```sql
-- Oracle / PostgreSQL 공통
ALTER TABLE 테이블명 ADD CONSTRAINT 제약조건이름 PRIMARY KEY (컬럼명);

-- 예제: 뒤늦게 PK 추가 
ALTER TABLE 사원 ADD CONSTRAINT pk_사원 PRIMARY KEY (사번); 

-- 예제: 뒤늦게 FK 추가 
ALTER TABLE 사원 ADD CONSTRAINT fk_부서 FOREIGN KEY (부서코드) REFERENCES 부서(부서코드);
```

### 컬럼 속성 변경 — ⚠️ DB별 문법 다름!

"글자 수 제한을 늘려주세요!" 할 때 쓴다.

| 구분            | 문법                                  |
| ------------- | ----------------------------------- |
| Oracle (SQLD) | `ALTER TABLE … MODIFY`              |
| PostgreSQL    | `ALTER TABLE … ALTER COLUMN … TYPE` |

#### SQLD / Oracle — `MODIFY`

```sql
-- 컬럼 데이터 타입 변경 (Oracle)
ALTER TABLE 테이블명
MODIFY (
    컬럼명 새로운_데이터타입
);

-- 예제
ALTER TABLE 사원
MODIFY (
    이름 VARCHAR2(100)
);
```

#### 실무 / PostgreSQL - `ALTER COLUMN … TYPE`

```sql
-- 컬럼 데이터 타입 변경 (PostgreSQL)
ALTER TABLE 테이블명
ALTER COLUMN 컬럼명
TYPE 새로운_데이터타입;

-- 예제
ALTER TABLE 사원
ALTER COLUMN 이름
TYPE VARCHAR(100);
```

---
## Code Core Points : 부수고 이름 바꾸기 (`DROP`, `TRUNCATE`, `RENAME`)

### 테이블 삭제 (`DROP TABLE`)

건물 자체를 다이너마이트로 폭파시킨다. 뼈대도, 안의 데이터도 흔적 없이 사라진다.

```sql
-- 기본
DROP TABLE 테이블명;

-- CASCADE: FK로 엮인 제약조건까지 강제로 끊고 삭제 (Oracle)
DROP TABLE 테이블명 CASCADE CONSTRAINTS;

-- PostgreSQL
DROP TABLE 테이블명 CASCADE;
```

> ⚠️ 뼈대(구조)와 데이터 **모두 삭제**. 복구 불가!

#### **CASCADE란?**

다른 테이블이 이 테이블을 FK로 참조하고 있으면 기본 `DROP TABLE`은 에러가 난다.
"누군가 이 테이블을 가리키고 있는데 그냥 날려버리면 어떡하냐!"고 DB가 거부하는 것이다.

```
부서 테이블 ◄── FK 참조 ── 사원 테이블

DROP TABLE 부서;  -- ❌ 에러! 사원 테이블이 참조 중
DROP TABLE 부서 CASCADE CONSTRAINTS;  -- ✅ FK 제약조건을 끊고 강제 삭제
```

단, CASCADE는 **제약조건(FK 규칙)만 끊는 것**이지 사원 테이블 자체를 지우는 게 아니다.
부서 테이블만 사라지고, 사원 테이블은 남아있지만 `부서코드` 컬럼의 FK 제약조건이 사라진 채로 남게 된다.

| 상황 | 결과 |
|------|------|
| FK 참조 없을 때 `DROP` | ✅ 정상 삭제 |
| FK 참조 있을 때 `DROP` | ❌ 에러 발생 |
| FK 참조 있을 때 `DROP CASCADE` | ✅ FK 제약조건 끊고 강제 삭제 |

> 💡 실무에서 CASCADE는 강력한 만큼 위험하다. 연쇄적으로 엮인 테이블이 많을수록
> 예상치 못한 데이터 정합성 문제가 생길 수 있으니 신중하게 사용해야 한다.


### 테이블 데이터 전체 초기화 (`TRUNCATE TABLE`)

건물 뼈대(구조)는 그대로 놔두고, 안의 데이터만 1초 만에 싹 다 비워버린다.
롤백 불가능해 DDL로 분류된다.

```sql
-- Oracle / PostgreSQL 공통
TRUNCATE TABLE 테이블명;

-- 예제
TRUNCATE TABLE 사원;
```

| 구분 | `DELETE` | `TRUNCATE` | `DROP` |
|------|---------|------------|--------|
| 데이터 | 조건부 삭제 | 전체 삭제 | 전체 삭제 |
| 구조 | 유지 | 유지 | 삭제 |
| 롤백 | ✅ 가능 | ❌ 불가 | ❌ 불가 |
| 속도 | 느림 | 빠름 | 빠름 |

### 테이블 이름 변경 (`RENAME`)

```sql
-- [SQLD / Oracle]
RENAME 기존테이블명 TO 새테이블명;

-- 예제
RENAME 사원 TO 직원;
```
```sql
-- [실무 / PostgreSQL]
ALTER TABLE 기존테이블명 RENAME TO 새테이블명;

-- 예제
ALTER TABLE 사원 RENAME TO 직원;
```