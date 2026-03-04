---
aliases:
  - 트랜잭션
  - ACID
  - savepoint
  - 세이브포인트
  - TCL
  - 커밋
  - 롤백
  - 동시성제어
tags:
  - SQL
  - CS
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DML_CRUD]]"
  - "[[SQL_DDL_Create]]"
---
# SQL TCL — 트랜잭션 제어 언어

## 개념 한 줄 요약

> **"트랜잭션(Transaction) 은 쪼갤 수 없는 최소한의 논리적 작업 단위이며, TCL 은 그 결과를 DB 에 영구 저장(`COMMIT`) 하거나 없던 일로 취소(`ROLLBACK`) 하는 제어 명령어다."**

---

## 왜 필요한가? — 은행 이체 시나리오

```
1. A 계좌에서 100만원 차감  (UPDATE)
2. ⚡ 서버 에러 발생!
3. B 계좌에 100만원 입금 코드 실행 안 됨

결과: A 의 돈은 사라지고 B 는 돈을 못 받음 → 돈이 증발
```

두 과정을 **하나의 트랜잭션으로 묶으면** 중간에 에러가 나도 `ROLLBACK` 으로 원상 복구할 수 있다.

---

## 트랜잭션 4가지 특징 — ACID ⭐️ SQLD 필수 암기 (원·일·고·지)

|약자|특징|설명|비유|
|---|---|---|---|
|**A**|**원**자성 Atomicity|All or Nothing. 모두 성공하거나 모두 실패|반반 치킨은 있어도 반반 송금은 없다|
|**C**|**일**관성 Consistency|트랜잭션 전후에 DB 제약조건이 유지되어야 함|법을 어기면서 송금할 순 없다|
|**I**|**고**립성 Isolation|실행 중인 트랜잭션에 다른 트랜잭션이 끼어들 수 없음|비밀번호 치는 동안 남이 내 키보드를 못 누른다|
|**D**|**지**속성 Durability|COMMIT 한 데이터는 서버가 꺼져도 영원히 보존|입금 확인증 받으면 은행 불나도 내 돈은 있다|

---

## 트랜잭션 흐름

```
① BEGIN            ← 트랜잭션 시작
② DML 실행         ← INSERT / UPDATE / DELETE
③ SAVEPOINT        ← (선택) 중간 북마크
④ COMMIT / ROLLBACK ← 확정 또는 취소 → Lock 해제
```

---

---

# ① BEGIN — 트랜잭션 시작

```sql
BEGIN;               -- PostgreSQL, MySQL
START TRANSACTION;   -- MySQL 표준 문법
-- Oracle 은 DML 을 치는 순간 자동으로 트랜잭션이 시작됨 (BEGIN 불필요)
```

> **Oracle vs PostgreSQL/MySQL** Oracle 은 `BEGIN` 없이 DML 을 치는 순간 자동으로 트랜잭션이 시작된다. PostgreSQL / MySQL 은 명시적으로 `BEGIN` 을 선언해줘야 한다.

---

# ② COMMIT — 영구 저장

메모리(Buffer) 에만 있던 변경 사항을 **DB(디스크)에 영구 반영** 한다.

```sql
BEGIN;
UPDATE 계좌 SET 잔액 = 잔액 - 10000 WHERE 이름 = '김맥북';
UPDATE 계좌 SET 잔액 = 잔액 + 10000 WHERE 이름 = '박옵시디언';
COMMIT;  -- ✅ 이제 진짜 저장. 이후엔 되돌릴 수 없다.
```

> COMMIT 이후에는 다른 사용자도 변경된 데이터를 조회할 수 있게 된다.

---

# ③ ROLLBACK — 전체 취소

마지막 COMMIT 시점으로 **모든 변경 사항을 되돌린다.**

```sql
BEGIN;
DELETE FROM 사원 WHERE 부서 = 'AI팀';
-- 앗, 잘못 지웠다!
ROLLBACK;  -- ✅ BEGIN 시점으로 전부 원상복구
```

---

---

# ④ SAVEPOINT — 중간 저장 북마크

전체를 되돌리는 게 아니라 **특정 시점까지만** 되돌리고 싶을 때 사용한다.

- `SAVEPOINT 포인트이름` : 현재 상태를 북마크로 임시 저장
- `ROLLBACK TO 포인트이름` : 딱 그 시점까지만 되돌아감

```sql
BEGIN;
INSERT INTO 영웅 (이름) VALUES ('전사');     -- 1번 작업
SAVEPOINT sp1;                              -- 📌 북마크 1
INSERT INTO 영웅 (이름) VALUES ('마법사');   -- 2번 작업
SAVEPOINT sp2;                              -- 📌 북마크 2
INSERT INTO 영웅 (이름) VALUES ('도적');     -- 3번 작업
ROLLBACK TO sp1;                            -- sp1 이후(마법사, 도적)만 취소
INSERT INTO 영웅 (이름) VALUES ('궁수');     -- 4번 작업 (롤백 후 새로 추가)
INSERT INTO 영웅 (이름) VALUES ('힐러');     -- 5번 작업
COMMIT;  -- 전사 + 궁수 + 힐러 최종 저장
```

## 실행 흐름 시각화

```
BEGIN
 │
 ▼
[전사 INSERT] ──► SAVEPOINT sp1 📌
 │
 ▼
[마법사 INSERT] ──► SAVEPOINT sp2 📌
 │
 ▼
[도적 INSERT]
 │
 ▼
ROLLBACK TO sp1
 └─ sp1 이후 작업(마법사, 도적) 취소
    → 전사는 유지됨
 │
 ▼
[궁수 INSERT]   ← 롤백 후 새로운 작업 계속 가능
 │
 ▼
[힐러 INSERT]
 │
 ▼
COMMIT
 └─ DB에 저장되는 것
    ✅ 전사
    ✅ 궁수
    ✅ 힐러
    ❌ 마법사 (롤백됨)
    ❌ 도적   (롤백됨)
```

> `ROLLBACK TO sp1` 이후에도 트랜잭션은 살아있다. 
> 새로운 작업을 이어서 할 수 있고, 마지막에 `COMMIT` 하면 **롤백 전 작업(전사) + 롤백 후 새 작업(궁수, 힐러)** 이 함께 저장된다.

---

---

# ⚡ Oracle DDL 의 암시적 COMMIT — SQLD 킬러 문항

> **"Oracle 에서 DDL 문장의 수행은 내부적으로 트랜잭션을 종료시킨다."** → [[SQL_DDL_Create]] 참고

## 핵심 규칙

|DB|DML 커밋 방식|DDL 커밋 방식|
|---|---|---|
|**Oracle**|수동 COMMIT 필요|**자동 COMMIT** (묵시적 실행)|
|**SQL Server**|수동 COMMIT 필요|수동 COMMIT 필요|
|**PostgreSQL**|수동 COMMIT 필요|수동 COMMIT 필요|

## SQLD 기출 문제 분석

```sql
-- 초기 상태: ID='001' VAL=100, ID='002' VAL=200
-- AUTO COMMIT = FALSE

UPDATE A SET VAL = 200 WHERE ID = '001';  -- DML: 미확정 상태
CREATE TABLE B (ID CHAR(3) PRIMARY KEY);  -- DDL 실행
ROLLBACK;
```

**Oracle 결과: ID='001' VAL = 200 (롤백 안 됨)**

```
① UPDATE 실행         → VAL 이 100 → 200 으로 변경 (미확정)
② CREATE TABLE 실행   → ❗ Oracle 이 DDL 전에 암시적 COMMIT 자동 실행
                         → UPDATE 가 확정(영구 저장)됨
③ ROLLBACK 실행       → 되돌릴 미확정 DML 이 없음 → 아무 효과 없음
결과: VAL = 200 (ROLLBACK 이 무의미해짐)
```

**SQL Server 결과: ID='001' VAL = 100 (롤백 됨)**

```
① UPDATE 실행         → VAL 이 100 → 200 으로 변경 (미확정)
② CREATE TABLE 실행   → DDL 도 트랜잭션 안에서 실행 (자동 COMMIT 없음)
③ ROLLBACK 실행       → UPDATE 와 CREATE TABLE 모두 취소
결과: VAL = 100 (ROLLBACK 이 정상 작동)
```

## 시각화

```
Oracle:
UPDATE(미확정) → CREATE TABLE → [자동 COMMIT 발생!] → ROLLBACK → 효과 없음
                                  ↑
                          여기서 UPDATE 가 확정됨

SQL Server:
UPDATE(미확정) → CREATE TABLE(미확정) → ROLLBACK → 둘 다 취소
```

## 추가 함정 — DML 중간에 DDL 을 치면?

```sql
-- Oracle 에서
INSERT INTO 사원 VALUES ('001', '홍길동');  -- 미확정
INSERT INTO 사원 VALUES ('002', '이순신');  -- 미확정
CREATE TABLE 임시 (id NUMBER);              -- DDL → 자동 COMMIT 발동!
-- 앞의 INSERT 2개도 같이 확정되어버림!
ROLLBACK;  -- 이미 늦었다. INSERT 는 취소 불가.
```

> ⚠️ Oracle 에서 실컷 DML 해놓고 `COMMIT` 안 한 상태에서 DDL 을 치면, 앞서 했던 DML 까지 자동 확정되어 되돌릴 수 없다. **DDL 은 트랜잭션을 종료시키는 폭탄이다.**

---

---

# 🔒 LOCK 주의 — COMMIT/ROLLBACK 을 너무 오래 안 하면?

DML 을 실행하면 해당 행(Row) 에 **Lock(잠금)** 이 걸린다. `COMMIT` 또는 `ROLLBACK` 으로 트랜잭션이 끝나야 Lock 이 해제된다.

```
사용자 A: BEGIN 후 UPDATE 만 하고 자리를 비움 (커피 타러...)

사용자 B: 같은 행을 UPDATE 시도
          │
          ▼
     🔒 Lock 대기 상태... (무한 대기)
```

|상황|결과|
|---|---|
|다른 사용자가 같은 행 수정 시도|COMMIT/ROLLBACK 전까지 무한 대기|
|대기가 너무 길어지면|Lock Timeout 에러 발생|
|여러 트랜잭션이 서로 물고 늘어지면|💀 DeadLock 발생|

> 실무에서 배치 작업이나 대량 UPDATE 후 깜빡하고 COMMIT 안 하면 다른 팀원의 작업이 통째로 멈출 수 있다. **DML 작업 후엔 반드시 바로 COMMIT 또는 ROLLBACK 으로 마무리하자.**

---

---

# 자주 하는 실수 & 함정

## "DBeaver·DataGrip 에서는 COMMIT 안 쳐도 자동 저장되던데요?"

툴의 기본 설정이 **Auto-Commit 모드** 이기 때문이다. 엔터 칠 때마다 즉시 COMMIT 이 된다. 하지만 터미널(`psql`, `sqlplus`) 에서는 직접 `COMMIT` 을 쳐야 하고, 창을 닫으면 날아간다.

## "DDL 도 실수하면 ROLLBACK 할 수 있죠?" ← SQLD 단골 문제 ⭐️

**Oracle 기준: 안 된다.** `CREATE` `DROP` `ALTER` 같은 DDL 은 실행 순간 **내부적으로 COMMIT 을 강제 실행** 해버린다. 앞서 했던 미확정 DML 까지 함께 확정되어 되돌릴 수 없다.

**PostgreSQL / SQL Server 기준: 가능하다.** DDL 도 트랜잭션 안에서 실행되므로 `ROLLBACK` 으로 취소할 수 있다.

---