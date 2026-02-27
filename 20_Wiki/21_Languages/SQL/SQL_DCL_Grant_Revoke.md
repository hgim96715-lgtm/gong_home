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
# 유저 권한 부여 및 회수 (DCL: Data Control Language)

## 한줄요약(Concept Summary)

DCL(Data Control Language)은 데이터베이스에 접근하는 유저(User)를 만들고, 그 유저가 특정 테이블을 보거나 수정할 수 있도록 권한을 주거나(`GRANT`), 줬던 권한을 다시 뺏는(`REVOKE`) 보안 및 접근 제어 명령어다.)

---
## Why (왜 필요한가)?

데이터베이스에는 회사의 명운이 걸린 민감한 고객 정보나 매출 데이터가 모두 들어있다. 만약 갓 입사한 인턴에게 무심코 `DELETE`나 `DROP` 권한이 있는 마스터 계정(슈퍼유저)을 줬다가, 터미널에서 실수로 운영 DB를 싹 다 날려버리면? 진짜 회사 문 닫아야 해. 그래서 "데이터 분석팀은 조회(`SELECT`)만 해!", "백엔드 개발팀은 데이터 삽입(`INSERT`)까지 해!"처럼 사람마다, 역할마다 철저하게 선을 긋고 안전망을 치기 위해 반드시 필요하다.

---
## Practical Context(실무맥락)

실무에서는 백엔드 서버(Spring, Node.js)가 DB에 접근할 전용 계정을 파주거나, 데이터 시각화 툴(Tableau, Metabase)을 연결할 때 '읽기 전용' 유저를 새로 세팅할 때 숨 쉬듯이 쓴다. **SQLD 시험**에서는 유저를 지울 때 생기는 연쇄 작용(`CASCADE`)과 비밀번호를 설정하는 기본 문법(`IDENTIFIED BY`)을 묻는 문제가 간혹 출제된다.

---
## Code Core Points: 계정 관리 3대장

계정을 만들고 지울 때, **SQLD(Oracle)와 실무(PostgreSQL)의 비밀번호 지정 문법**이 다르니 주의!

### 유저 생성 (`CREATE USER`) — 비밀번호 지정 문법이 DB마다 다름!

```sql
-- [SQLD / Oracle] IDENTIFIED BY
CREATE USER 신입사원 IDENTIFIED BY '비밀번호123';

-- [실무 / PostgreSQL] WITH PASSWORD
CREATE USER 신입사원 WITH PASSWORD '비밀번호123';
```

---

### 유저 수정 (`ALTER USER`) — 비밀번호 변경 & 계정 잠금/해제

비밀번호를 까먹었거나 변경 주기가 도래했을 때 사용한다.

```sql
-- [SQLD / Oracle] 비밀번호 변경: IDENTIFIED BY
ALTER USER 신입사원 IDENTIFIED BY '새로운비번456';

-- [실무 / PostgreSQL] 비밀번호 변경: WITH PASSWORD
-- ⚠️ Oracle의 IDENTIFIED BY 쓰면 에러! PostgreSQL은 WITH PASSWORD를 써야 함
ALTER USER 신입사원 WITH PASSWORD '새로운비번456';
```
```sql
-- 💡 계정 잠그기 / 풀기 (Oracle 기준)
ALTER USER 신입사원 ACCOUNT LOCK;    -- 잠금
ALTER USER 신입사원 ACCOUNT UNLOCK;  -- 잠금 해제
```

---

### 유저 삭제 (`DROP USER`)

퇴사한 직원의 계정을 DB에서 완전히 삭제한다.

```sql
-- 기본: 소유한 객체가 없을 때만 가능
DROP USER 신입사원;

-- [SQLD / Oracle] CASCADE: 유저 + 소유 객체 전부 삭제
DROP USER 신입사원 CASCADE;

-- [실무 / PostgreSQL] 2단계로 나눠서 삭제
DROP OWNED BY 신입사원;  -- 1단계: 소유 테이블 먼저 삭제
DROP USER 신입사원;      -- 2단계: 유저 삭제
```

#### ⚠️ 유저 삭제 시 치명적 함정

유저가 테이블을 만들어놓은 상태에서 그냥 `DROP USER`를 치면?

>❌ "이 유저가 소유한 객체가 있어서 삭제할 수 없습니다!" 에러 발생

| 상황 | Oracle | PostgreSQL |
|------|--------|------------|
| 소유 객체 없을 때 | `DROP USER 유저명` | `DROP USER 유저명` |
| 소유 객체 있을 때 | `DROP USER 유저명 CASCADE` | `DROP OWNED BY 유저명` → `DROP USER 유저명` |
| CASCADE 의미 | 유저 + 테이블 + 데이터 **한 번에 폭파** | PostgreSQL은 2단계로 나눠서 처리 |

>  Oracle `CASCADE`는 강력한 만큼 위험하다.
> PostgreSQL이 2단계로 나눈 이유도 실수로 전부 날리는 걸 방지하기 위해서다.

---
## Code Core Points: DCL 3대장 (`GRANT`, `REVOKE`, `ROLE`)

권한의 종류부터 구분하고 가자.

| 권한 종류 | 설명 | 예시 |
|---------|------|------|
| **시스템 권한** | DB 자체를 건드리는 굵직한 권한 | 로그인, 테이블 생성 |
| **객체 권한** | 특정 테이블에 대한 권한 | 특정 테이블 조회, 수정 |

---

### 권한 부여 (`GRANT`)

```sql
-- 문법 구조
GRANT 권한 ON 객체 TO 유저;
--           ↑ 시스템 권한은 ON 객체 없이 바로 TO 유저
```
```sql
-- [시스템 권한] DB 접속 권한 부여
GRANT CREATE SESSION TO 신입사원;

-- [객체 권한] 특정 테이블 조회 권한 부여
GRANT SELECT ON 사원 TO 신입사원;

-- [킬러 옵션] WITH GRANT OPTION  SQLD 단골 기출
-- 권한을 받으면서, 그 권한을 남에게 나눠줄 수 있는 권한까지 받음
GRANT SELECT ON 사원 TO 신입사원 WITH GRANT OPTION;
```

>  **WITH GRANT OPTION 주의!**
> 신입사원이 WITH GRANT OPTION으로 받은 권한을 다른 사람에게 줬을 때,
> 신입사원의 권한을 `REVOKE`하면 신입사원이 나눠준 권한도 **연쇄적으로 회수**된다.

---

### 권한 회수 (`REVOKE`)

```sql
-- 문법 구조
REVOKE 권한 ON 객체 FROM 유저;

-- 사원 테이블 조회 권한 회수
REVOKE SELECT ON 사원 FROM 신입사원;
```

> 💡 **GRANT vs REVOKE 문법 비교**

| 구분 | 방향 키워드 |
|------|-----------|
| `GRANT` (줄 때) | `TO` 유저 |
| `REVOKE` (뺏을 때) | `FROM` 유저 |

---
### 롤 (`ROLE`) — 권한 바구니

100명에게 10개씩 권한을 `GRANT` 하면 1000번을 쳐야 한다. 
ROLE은 이걸 한 번에 해결한다.

>권한들 → ROLE(바구니)에 담기 → 유저에게 바구니째로 전달

```sql
-- 1단계: 빈 바구니(ROLE) 생성
CREATE ROLE 데이터분석가;

-- 2단계: 바구니 안에 권한 담기
GRANT SELECT ON 사원 TO 데이터분석가;
GRANT SELECT ON 매출 TO 데이터분석가;
GRANT SELECT ON 고객 TO 데이터분석가;

-- 3단계: 유저에게 바구니째로 전달
GRANT 데이터분석가 TO 신입사원A;
GRANT 데이터분석가 TO 신입사원B;
-- 앞으로 신입사원이 100명 와도 마지막 줄 한 번만 추가하면 끝!
```

| 방식 | 필요한 GRANT 횟수 |
|------|----------------|
| 직접 부여 (100명 × 10개 권한) | 1,000번 |
| ROLE 사용 (100명 × 바구니 1개) | 100번 + 권한 설정 몇 번 |



