---
aliases:
  - DCL
  - 데이터 제어어
  - GRANT
  - REVOKE
  - ROLE
  - 권한 관리
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DDL_Create]]"
  - "[[SQL_DML_CRUD]]"
---
# SQL_DCL_Grant_Revoke

## 개념 한 줄 요약

> **"누가 무엇을 할 수 있는지 권한을 부여하고 회수하는 명령어."** DCL(Data Control Language) 은 데이터베이스의 보안과 접근 제어를 담당한다.

---

---

# ① 권한의 2가지 종류

```
시스템 권한 (System Privilege)
  → DB 전체 구조에 영향을 주는 권한
  → 예) CREATE SESSION (접속), CREATE TABLE (테이블 생성), CREATE USER

객체 권한 (Object Privilege)
  → 특정 테이블/뷰 등 개별 객체에 대한 권한
  → 예) SELECT, INSERT, UPDATE, DELETE ON 특정테이블
```

---

---

# ② GRANT — 권한 부여

```sql
-- 시스템 권한: DB 접속 권한 부여
GRANT CREATE SESSION TO 신입사원;

-- 객체 권한: 특정 테이블 조회 권한 부여
GRANT SELECT ON 사원 TO 신입사원;

-- 여러 권한을 한 번에 부여
GRANT SELECT, INSERT, UPDATE ON 사원 TO 신입사원;
```

---

---

# ③ WITH GRANT OPTION ⭐ (객체 권한 전용)

## 뭔가요?

> 권한을 받을 때 **"이 권한을 다른 사람에게 다시 줄 수 있는 능력"** 까지 함께 받는 옵션. 
> 객체 권한(SELECT, INSERT 등)에만 사용 가능.

```sql
-- 일반 GRANT
GRANT SELECT ON 사원 TO 신입사원;

-- WITH GRANT OPTION
GRANT SELECT ON 사원 TO 신입사원 WITH GRANT OPTION;
```

## 차이

```
일반 GRANT:
  관리자 ──▷ 신입사원 (SELECT 가능)
  신입사원 ──▷ 인턴에게 줄 수 없음 ❌

WITH GRANT OPTION:
  관리자 ──▷ 신입사원 (SELECT 가능 + 다른 사람에게 줄 수도 있음)
  신입사원 ──▷ 인턴 (SELECT 줄 수 있음 ✅)
```

## ⚠️ REVOKE 하면 연쇄 회수된다

```
부여 흐름:
  관리자 ──(WITH GRANT OPTION)──▷ 신입사원 ──▷ 인턴

관리자가 신입사원 권한을 REVOKE 하면:
  신입사원 권한 회수 ✅
  인턴 권한도 자동 연쇄 회수 ✅  ← 핵심 포인트!
```

---

---

# ④ WITH ADMIN OPTION ⭐ (시스템 권한 전용)

## 뭔가요?

> WITH GRANT OPTION 과 같은 개념이지만 **시스템 권한 전용** 옵션. 
> 시스템 권한을 받으면서 **"이 권한을 다른 사람에게 다시 줄 수 있는 능력"** 까지 함께 받는다.

```sql
-- WITH ADMIN OPTION
GRANT CREATE SESSION TO 신입사원 WITH ADMIN OPTION;
```

```
WITH ADMIN OPTION:
  관리자 ──(WITH ADMIN OPTION)──▷ 신입사원 (CREATE SESSION 가능 + 다른 사람에게 줄 수도 있음)
  신입사원 ──▷ 인턴 (CREATE SESSION 줄 수 있음 ✅)
```

## ⚠️ REVOKE 해도 연쇄 회수 안 된다

```
부여 흐름:
  관리자 ──(WITH ADMIN OPTION)──▷ 신입사원 ──▷ 인턴

관리자가 신입사원 권한을 REVOKE 하면:
  신입사원 권한 회수 ✅
  인턴 권한은 그대로 남음 ❌  ← GRANT OPTION 과 다른 핵심 포인트!
```

---

---

# ⑤ WITH GRANT OPTION vs WITH ADMIN OPTION ⭐ SQLD 단골 비교

|구분|WITH GRANT OPTION|WITH ADMIN OPTION|
|---|:-:|:-:|
|**적용 대상**|객체 권한 (SELECT, INSERT ...)|시스템 권한 (CREATE SESSION ...)|
|**재부여 가능**|✅|✅|
|**REVOKE 시 연쇄 회수**|✅ 연쇄 회수됨|❌ 연쇄 회수 안 됨|

```
암기 포인트:
GRANT OPTION  →  객체 권한  →  연쇄 회수 O (엄격)
ADMIN OPTION  →  시스템 권한  →  연쇄 회수 X (느슨)
```

---

---

# ⑥ REVOKE — 권한 회수

```sql
-- 객체 권한 회수
REVOKE SELECT ON 사원 FROM 신입사원;

-- 시스템 권한 회수
REVOKE CREATE SESSION FROM 신입사원;

-- CASCADE: 연쇄 회수 (WITH GRANT OPTION 으로 퍼진 권한까지 전부)
REVOKE SELECT ON 사원 FROM 신입사원 CASCADE;
```

---

---

# ⑦ ROLE — 권한 묶음

> 여러 권한을 하나의 ROLE 로 묶어서 한 번에 부여/회수한다. 사용자가 많을수록 관리가 훨씬 편해진다.

```sql
-- ROLE 생성
CREATE ROLE 신입사원_ROLE;

-- ROLE 에 권한 담기
GRANT CREATE SESSION TO 신입사원_ROLE;
GRANT SELECT ON 사원 TO 신입사원_ROLE;
GRANT SELECT ON 부서 TO 신입사원_ROLE;

-- 사용자에게 ROLE 한 방에 부여
GRANT 신입사원_ROLE TO 홍길동;
GRANT 신입사원_ROLE TO 김철수;

-- ROLE 회수
REVOKE 신입사원_ROLE FROM 홍길동;
```

```
ROLE 장점:
사용자 100명에게 권한 10개를 부여할 때
→ 직접 부여: 100 × 10 = 1000번
→ ROLE 사용: ROLE 에 10개 + 100명에게 1번씩 = 110번
```

---

---

# 전체 흐름 정리

```
           시스템 권한                      객체 권한
           (CREATE SESSION 등)             (SELECT ON 테이블 등)
                │                                │
         WITH ADMIN OPTION               WITH GRANT OPTION
                │                                │
         재부여 가능 ✅                    재부여 가능 ✅
                │                                │
         연쇄 회수 안 됨 ❌                연쇄 회수 됨 ✅
```

---

---

# SQLD 시험 출제 패턴

**패턴 ① "WITH GRANT OPTION 으로 부여된 권한을 REVOKE 하면?"** → 중간 사용자가 다른 사람에게 준 권한도 **연쇄 회수**된다.

**패턴 ② "WITH ADMIN OPTION 으로 부여된 권한을 REVOKE 하면?"** → 중간 사용자 권한만 회수, 이미 퍼진 권한은 **회수되지 않는다**.

**패턴 ③ "다음 중 객체 권한에 해당하는 것은?"** → `SELECT ON 테이블` → 객체 권한 ✅ → `CREATE SESSION` → 시스템 권한 ✅

---
