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

#  (TCL: Transaction Control Language) 트랜잭션을 제어하는 명령어 

## 개념 한 줄 요약

트랜잭션(Transaction)은 데이터베이스의 상태를 변화시키는 '쪼갤 수 없는 최소한의 논리적 작업 단위'이며, 
TCL(Transaction Control Language)은 이 작업의 결과를 DB에 영구 저장(`COMMIT`)하거나 아예 없던 일로 취소(`ROLLBACK`)하는 제어 명령어이다.

---
## 왜 필요한가? (Why)

**은행 이체 시나리오**

1. A 계좌에서 100만원을 뺀다 `(UPDATE)`
2. ⚡️ 서버 에러 발생!
3. B 계좌에 100만원을 넣는 코드는 실행되지 못함
4. **결과:** A의 돈은 사라지고 B는 돈을 못 받음 → 돈이 증발

이 두 과정을 **하나의 트랜잭션**으로 묶으면, 중간에 에러가 나도 1번까지 `ROLLBACK`해서 원상 복구할 수 있다.



---
## 트랜잭션의 4가지 핵심 특징 (ACID) - SQLD 필수 암기!(암기: 원.일.고.지)

| 약자    | 특징                  | 설명                                 | 비유                         |
| ----- | ------------------- | ---------------------------------- | -------------------------- |
| **A** | **원자성** Atomicity   | All or Nothing. 모두 성공하거나 모두 실패해야 함 | 반반 치킨은 있어도 반반 송금은 없다       |
| **C** | **일관성** Consistency | 트랜잭션 전후에 DB의 제약조건이 유지되어야 함         | 법을 어기면서 송금할 순 없다           |
| **I** | **고립성** Isolation   | 실행 중인 트랜잭션에 다른 트랜잭션이 끼어들 수 없음      | 비밀번호 치는 동안 남이 내 키보드를 못 누른다 |
| **D** | **지속성** Durability  | Commit한 데이터는 서버가 꺼져도 영원히 보존됨       | 입금 확인증 받으면 은행 불나도 내 돈은 있다  |

---
## 기본 명령어

**TCL (Transaction Control Language)** 은 DML(Insert, Update, Delete)을 확정하거나 취소하는 명령어입니다.

### ① BEGIN (트랜잭션 시작)

"지금부터 이 작업들을 하나의 묶음으로 처리하겠다!"

```sql
BEGIN;  -- PostgreSQL, MySQL
START TRANSACTION;  -- MySQL 표준 문법
-- Oracle은 DML을 치는 순간 자동으로 트랜잭션이 시작됨
```

> Oracle은 `BEGIN` 없이 DML을 치는 순간 자동으로 트랜잭션이 시작된다.
> PostgreSQL / MySQL은 명시적으로 `BEGIN`을 선언해줘야 한다.

```sql
① BEGIN          ← 트랜잭션 시작
② DML 실행       ← INSERT / UPDATE / DELETE
③ SAVEPOINT      ← (선택) 중간 북마크
④ COMMIT / ROLLBACK  ← 확정 또는 취소 → Lock 해제
```


### ② COMMIT — 영구 저장

메모리(Buffer)에만 있던 변경 사항을 **DB(디스크)에 영구 반영**한다.

```sql
BEGIN;
UPDATE 계좌 SET 잔액 = 잔액 - 10000 WHERE 이름 = '김맥북';
UPDATE 계좌 SET 잔액 = 잔액 + 10000 WHERE 이름 = '박옵시디언';
COMMIT; -- ✅ 이제 진짜 저장. 이후엔 되돌릴 수 없다.
```

>Commit 이후에는 다른 사용자도 변경된 데이터를 조회할 수 있게 된다.

### ③ ROLLBACK — 전체 취소

마지막 Commit 시점으로 **모든 변경 사항을 되돌린다.**

```sql
BEGIN;
DELETE FROM 사원 WHERE 부서 = 'AI팀';
-- 앗, 잘못 지웠다!
ROLLBACK; -- ✅ BEGIN 시점으로 전부 원상복구
```

---
##  SAVEPOINT — 중간 저장 북마크

전체를 되돌리는 게 아니라 **특정 시점까지만** 되돌리고 싶을 때 사용한다.

- **`SAVEPOINT 포인트이름;`** : 현재까지의 작업 상태를 '포인트이름'이라는 북마크로 임시 저장한다.
- **`ROLLBACK TO 포인트이름;`** : 전체를 다 취소하는 게 아니라, 내가 지정했던 딱 그 임시 저장 시점까지만 되돌아간다.

```sql
BEGIN;
INSERT INTO 영웅 (이름) VALUES ('전사');    -- 1번 작업
SAVEPOINT sp1;                             -- 📌 북마크 1 저장
INSERT INTO 영웅 (이름) VALUES ('마법사');  -- 2번 작업
SAVEPOINT sp2;                             -- 📌 북마크 2 저장
INSERT INTO 영웅 (이름) VALUES ('도적');    -- 3번 작업
ROLLBACK TO sp1;                           -- sp1 이후(마법사, 도적)만 취소
INSERT INTO 영웅 (이름) VALUES ('궁수'); -- 4번 작업 (롤백 후 새로 추가)
INSERT INTO 영웅 (이름) VALUES ('힐러'); -- 5번 작업
COMMIT; -- 전사 + 궁수 + 힐러 최종 저장
```

### **실행 흐름 시각화**

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
    ✔ 전사
    ✔ 궁수
    ✔ 힐러

```

>`ROLLBACK TO sp1` 이후에도 트랜잭션은 살아있다.
> 새로운 작업을 이어서 할 수 있고, 마지막에 `COMMIT`하면 **롤백 전 작업(전사) + 롤백 후 새 작업(궁수, 힐러)** 이 함께 저장된다.
> 마법사와 도적처럼 sp1 이후에 있던 작업만 사라진다.

---
## ⚠️ LOCK 주의 — Commit/Rollback을 너무 오래 안 하면?

DML을 실행하면 해당 행(Row)에 **Lock(잠금)** 이 걸린다. 
`COMMIT` 또는 `ROLLBACK`으로 트랜잭션이 끝나야 Lock이 해제된다.

```sql
사용자 A가 BEGIN 후 UPDATE만 하고 자리를 비움 (커피 타러...)

사용자 B가 같은 행을 UPDATE 시도
         │
         ▼
    🔒 Lock 대기 상태... (무한 대기)
```

### Lock이 오래 지속되면 발생하는 문제들

| 상황                  | 결과                        |
| ------------------- | ------------------------- |
| 다른 사용자가 같은 행 수정 시도  | COMMIT/ROLLBACK 전까지 무한 대기 |
| 대기가 너무 길어지면         | Lock Timeout 에러 발생        |
| 여러 트랜잭션이 서로 물고 늘어지면 | 💀 DeadLock 발생            |

>실무에서 배치 작업이나 대량 UPDATE 후 깜빡하고 Commit 안 하면 다른 팀원의 작업이 통째로 멈출 수 있다. 
>**DML 작업 후엔 반드시 바로 COMMIT 또는 ROLLBACK으로 마무리하자.**

---
## 자주 하는 실수 & 함정

**"DBeaver나 DataGrip에서는 COMMIT 안 쳐도 자동 저장되던데요?"**

툴의 기본 설정이 **Auto-Commit** 모드이기 때문이다. 
엔터 칠 때마다 즉시 Commit이 되어버린다. 하지만 터미널(`psql`, `sqlplus`)에서는 직접 `COMMIT`을 쳐야 하고, 창을 닫으면 날아간다.

**"DDL도 실수하면 ROLLBACK 할 수 있죠?" ← SQLD 단골 문제 🌟**

안 된다. **Oracle 기준**으로 `CREATE`, `DROP`, `ALTER` 같은 DDL은 실행 순간 **내부적으로 COMMIT을 강제 실행**해버린다. 
`INSERT` 실컷 해놓고 `COMMIT` 안 한 상태에서 `CREATE TABLE`을 치면, 앞서 했던 `INSERT`까지 자동 확정되어 되돌릴 수 없다.