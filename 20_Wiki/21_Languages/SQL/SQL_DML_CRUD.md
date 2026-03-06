---
aliases:
  - 데이터 조작
  - INSERT
  - UPDATE
  - DELETE
  - MERGE
  - CRUD
tags:
  - SQL
related:
  - "[[SQL_DDL_Create]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
  - "[[Python_Database_Connect]]"
  - "[[SQL_Keys_and_Identifiers]]"
  - "[[SQL_Database_Transactions_TCL]]"
  - "[[SQL_DCL_Grant_Revoke]]"
---
# SQL DML — 데이터 넣고, 고치고, 지우고, 병합하기

## 개념 한 줄 요약

> **"DML(Data Manipulation Language) 은 뼈대만 있는 테이블 안에 실제 데이터(행)를 생성(INSERT), 수정(UPDATE), 삭제(DELETE), 병합(MERGE) 하는 데이터 조작 명령어다."**

---

## 왜 필요한가?

유저가 '가입하기' 버튼을 눌렀을 때 정보를 저장하고(INSERT), 비밀번호를 바꾸면 수정하고(UPDATE), 탈퇴하면 지워야(DELETE) 한다. 껍데기뿐인 DB 에 생명력을 불어넣고 실제 서비스를 굴러가게 만드는 핵심 엔진이다.

> 실무 데이터 엔지니어링에서는 파이프라인(ETL) 을 통해 매일 수백만 건의 데이터를 쏟아붓기 때문에 `INSERT INTO ... SELECT` 나 `MERGE`(Upsert) 구문을 숨 쉬듯이 쓴다.

---

## DML 과 트랜잭션 — 생명줄

DML 은 실행한다고 바로 DB 에 영구 반영되는 게 아니다. **트랜잭션(Transaction)** 안에서 동작하기 때문에 반드시 `COMMIT` 을 해줘야 최종 저장된다.

```sql
UPDATE 사원 SET 연봉 = 9999 WHERE 사번 = 101;
-- 아직 DB에 반영 안 됨! 나만 보이는 임시 상태

COMMIT;    -- ✅ 이제 진짜 저장됨
ROLLBACK;  -- ↩️ COMMIT 전이라면 되돌리기 가능
```

|DB|기본 동작|설명|
|---|---|---|
|**SQL Server**|✅ Auto Commit|DML 실행 즉시 자동 반영. `BEGIN TRAN` 선언해야 롤백 가능|
|**Oracle**|❌ 수동 Commit|명시적으로 `COMMIT` 해줘야 반영됨|
|**PostgreSQL**|❌ 수동 Commit|Oracle 과 동일. 단, psql·DBeaver 는 기본 Auto Commit|

> **PostgreSQL 주의!** DBeaver 나 psql 같은 GUI·터미널은 기본 **Auto Commit 모드** 다. 중요한 작업 전엔 반드시 `BEGIN;` 으로 트랜잭션을 열어줘야 롤백이 가능하다.

```sql
BEGIN;
DELETE FROM 사원 WHERE 사번 = 101;
-- SELECT 로 조건 확인 후...
ROLLBACK;  -- 또는 COMMIT;
```

> COMMIT · ROLLBACK · SAVEPOINT 상세 → [[SQL_Database_Transactions_TCL]] 참고

---

---

# ① INSERT — 데이터 넣기

## A. 정석 — 원하는 컬럼만 지정 (권장)

```sql
INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);

-- 예제
INSERT INTO 사원 (사번, 이름, 부서명)
VALUES (101, '김맥북', '데이터엔지니어링팀');
-- 지정하지 않은 컬럼은 자동으로 NULL 또는 DEFAULT 처리
```

## B. 생략형 — 컬럼명 없이 전체 삽입

```sql
INSERT INTO 테이블명 VALUES (값1, 값2, 값3, ...);

-- 예제
INSERT INTO 사원
VALUES (102, '박옵시디언', '기획팀', '대리', 3000);
-- ⚠️ 모든 컬럼을 만들어진 순서 그대로 빠짐없이 채워야 함
```

## C. 대량 삽입 — 다른 테이블에서 복사

```sql
INSERT INTO 타겟테이블 (컬럼1, 컬럼2)
SELECT 컬럼A, 컬럼B FROM 소스테이블 WHERE 조건;

-- 예제
INSERT INTO 우수사원 (사번, 이름)
SELECT 사번, 이름 FROM 사원 WHERE 실적 > 90;
-- VALUES 없이 SELECT 결과를 통째로 삽입
```

## 방식 한눈에 비교

| 방식    | 컬럼 지정   | VALUES | 주의사항               |
| ----- | ------- | ------ | ------------------ |
| 정석    | ✅ 일부 지정 | ✅      | 미지정 컬럼 → NULL      |
| 생략형   | ❌ 생략    | ✅      | 개수·순서 완벽히 일치       |
| 대량 삽입 | ✅ 일부 지정 | ❌      | SELECT 결과와 컬럼 수 일치 |

---

## ⚠️ INSERT 에러가 나는 상황

**상황 1 — NOT NULL 컬럼을 비웠을 때**

A 방식으로 일부 컬럼만 지정하면 나머지는 `NULL` 이 들어간다. 해당 컬럼에 `PK` 또는 `NOT NULL` 제약조건이 걸려 있다면 즉시 에러.

```sql
-- SAMPLE 테이블: COL1(PK), COL2, COL3 DEFAULT 'D'
INSERT INTO SAMPLE (COL2, COL3) VALUES ('S', 'A');
-- COL1 을 지정 안 함 → NULL 삽입 시도 → PK 위반 → ❌ 에러!
```

**상황 2 — 생략형에서 값 개수가 안 맞을 때**

컬럼명을 생략하면 DB 는 "전체 컬럼에 순서대로 전부 넣겠다" 고 해석한다. 값이 하나라도 부족하거나 많으면 즉시 에러.

```
테이블 컬럼 수 = VALUES 안의 값 개수  ← 이 공식이 반드시 성립해야 한다
```

---

## DEFAULT 는 언제 작동하나?

|상황|결과|
|---|---|
|컬럼을 아예 지정 안 함|DEFAULT 값이 자동 삽입|
|VALUES 에 값을 직접 명시|DEFAULT 무시, 명시한 값 삽입|

---

## SQLD 표기법 — 테이블 다이어그램에서 PK 구분하기

```
┌─────────────────┐
│      COL1       │  ← 선 위쪽 단독 = PK
├─────────────────┤
│ COL2            │
│ COL3 DEFAULT 'D'│  ← 선 아래쪽 = 일반 컬럼
└─────────────────┘
```

> 가로선 **위쪽에 단독으로 있는 컬럼 = PK** PK 는 자동으로 `NOT NULL + UNIQUE` 조건을 가진다.

---

## Python 에서 INSERT

실무에서는 Python 코드에서 데이터를 밀어넣는 방식을 많이 쓴다.

```python
cursor.execute("""
    INSERT INTO user_logs (user_name, price, timestamp)
    VALUES (%s, %s, NOW())
""", (data["user_name"], data["price"]))
# NOW()는 SQL 안에 쓰니까 파이썬 튜플에서 빠지는 게 포인트!
```

> Python DB 연결 상세 → [[Python_Database_Connect]] 참고

---

---

# ② UPDATE — 데이터 수정하기

> 🚨 **반드시 `WHERE` 조건을 붙여야 한다. 안 그러면 모든 행이 다 바뀐다.**

```sql
UPDATE 테이블명
SET 컬럼1 = 값1, 컬럼2 = 값2
WHERE 조건;
```

```sql
-- 사번 101 직원의 부서를 'AI팀', 연봉을 5000 으로 변경
UPDATE 사원
SET
    부서명 = 'AI팀',
    연봉   = 5000
WHERE 사번 = 101;

-- ❌ WHERE 절 없이 실행하면?
-- 회사 전 직원의 부서가 AI팀, 연봉이 5000 으로 덮어써지는 대참사!
```

---

---

# ③ DELETE — 데이터 삭제하기

> 🚨 **`WHERE` 없이 쓰면 테이블이 텅 빈다.**

```sql
DELETE FROM 테이블명
WHERE 조건;
```

```sql
-- 사번 102 직원 삭제
DELETE FROM 사원
WHERE 사번 = 102;
-- DELETE 뒤에 컬럼명을 적지 않는다. 행(Row) 전체를 날리는 명령이기 때문.
```

> DELETE vs TRUNCATE 비교 → [[SQL_DDL_Create#④ DROP · TRUNCATE · DELETE 비교|DELETE vs TRUNCATE]] 참고

---

---

# ④ MERGE — 엎어치기 (Upsert)

> **"데이터가 이미 있으면 UPDATE, 없으면 INSERT 해라!" — 실무 끝판왕** 중복 에러(PK 위반)를 피하면서 최신 데이터를 유지할 때 필수로 사용한다.

---

## A. Oracle (SQLD 표준) — MERGE INTO

`ON` 절에서 기준 컬럼을 비교한 뒤, 결과에 따라 두 갈래로 처리한다.

```sql
MERGE INTO 타겟테이블 T
USING 소스테이블 S
ON (T.사번 = S.사번)              -- 두 테이블의 사번을 비교해서 매칭 여부 확인

WHEN MATCHED THEN                 -- 일치하는 행이 있을 때 (기존 사원)
    UPDATE SET
        T.부서명 = S.부서명,
        T.연봉   = S.연봉
    -- ✅ UPDATE 절 안에서 타겟 테이블 이름(T)을 다시 쓰지 않고 바로 SET 으로 시작

WHEN NOT MATCHED THEN             -- 일치하는 행이 없을 때 (신입 사원)
    INSERT (사번, 이름, 부서명)
    VALUES (S.사번, S.이름, S.부서명);
    -- ✅ INSERT INTO 타겟테이블 이 아니라 그냥 INSERT — 맨 위 MERGE INTO 에서 이미 선언했기 때문
```

### WHEN MATCHED 에서 조건 추가 — SQLD 빈출

```sql
WHEN MATCHED THEN
    UPDATE SET T.연봉 = S.연봉
    WHERE S.실적 > 90  -- 매칭됐어도 실적 90 초과인 경우만 업데이트
```

> `UPDATE SET ...` 뒤에 `WHERE` 를 추가해서 더 세밀하게 조건을 걸 수 있다.

---

## B. PostgreSQL (실무) — ON CONFLICT

일단 `INSERT` 를 시도한 다음 **충돌(Conflict)이 나면 어떻게 할지** 를 정하는 방식. Oracle 보다 훨씬 간결하다.

```sql
INSERT INTO 테이블명 (사번, 이름, 부서명)
VALUES (101, '김맥북', 'AI팀')

ON CONFLICT (사번)           -- 사번(PK)이 충돌나면?
DO UPDATE SET
    부서명 = EXCLUDED.부서명,
    연봉   = EXCLUDED.연봉;
    -- EXCLUDED = INSERT 하려다 충돌 나서 튕겨 나온 새 데이터
    -- Oracle 의 소스테이블 S 와 완전히 같은 역할
```

---

## Oracle vs PostgreSQL MERGE 비교

|구분|Oracle `MERGE INTO`|PostgreSQL `ON CONFLICT`|
|---|---|---|
|방식|소스·타겟 명시적 분리|INSERT 시도 후 충돌 처리|
|없을 때|`WHEN NOT MATCHED THEN INSERT`|기본 INSERT|
|있을 때|`WHEN MATCHED THEN UPDATE`|`ON CONFLICT DO UPDATE`|
|새 데이터 참조|`S.컬럼명` (소스 별칭)|`EXCLUDED.컬럼명`|

---

## ⚠️ MERGE 자주 하는 실수

**Oracle MERGE 안의 INSERT 에 SELECT 를 쓰면?**

```sql
-- ❌ 절대 안 됨!
WHEN NOT MATCHED THEN
    INSERT (사번) SELECT S.사번 FROM ...  -- 에러

-- ✅ VALUES 만 사용 가능
WHEN NOT MATCHED THEN
    INSERT (사번) VALUES (S.사번)
```

`USING 소스테이블 S` 에서 이미 SELECT 로 데이터를 퍼왔기 때문에, INSERT 절 안에서 다시 SELECT 를 쓸 수 없다.

---

---