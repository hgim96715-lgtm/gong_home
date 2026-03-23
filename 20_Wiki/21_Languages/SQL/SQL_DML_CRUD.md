---
aliases:
  - 데이터 조작
  - INSERT
  - UPDATE
  - DELETE
  - MERGE
  - CRUD
  - UPSERT
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
# SQL_DML_CRUD — 데이터 넣고 고치고 지우기

## 한 줄 요약

```
INSERT  → 행 추가
UPDATE  → 행 수정
DELETE  → 행 삭제
MERGE   → 있으면 UPDATE / 없으면 INSERT (UPSERT)
```

---

---

# ① DML 과 트랜잭션

```
DML 실행 → 즉시 DB 에 반영되지 않음
트랜잭션 안에서 동작 → COMMIT 해야 최종 저장
```

```sql
UPDATE 사원 SET 연봉 = 9999 WHERE 사번 = 101;
-- 아직 임시 상태 (나만 보임)

COMMIT;    -- 진짜 저장
ROLLBACK;  -- 되돌리기 (COMMIT 전에만 가능)
```

|DB|기본 동작|
|---|---|
|Oracle|수동 COMMIT 필요|
|PostgreSQL|수동 COMMIT 필요 (psql/DBeaver 는 AutoCommit)|
|SQL Server|AutoCommit (BEGIN TRAN 선언해야 롤백 가능)|

```
⚠️ PostgreSQL + DBeaver / psql:
  기본 AutoCommit 모드
  실수로 DELETE 해도 되돌리려면 먼저 BEGIN; 으로 트랜잭션 열기

  BEGIN;
  DELETE FROM 사원 WHERE 사번 = 101;
  -- 확인 후 ROLLBACK; 또는 COMMIT;
```

---

---

# ② INSERT — 데이터 넣기

## 컬럼 지정 방식 (권장)

```sql
INSERT INTO 사원 (사번, 이름, 부서명)
VALUES (101, '김철수', '데이터팀');
-- 지정 안 한 컬럼 → NULL 또는 DEFAULT 처리
```

## 전체 컬럼 방식 (순서 일치 필수)

```sql
INSERT INTO 사원
VALUES (102, '이영희', '기획팀', '대리', 3000);
-- 모든 컬럼을 생성 순서대로 빠짐없이
```

## 다른 테이블에서 복사

```sql
INSERT INTO 우수사원 (사번, 이름)
SELECT 사번, 이름 FROM 사원 WHERE 실적 > 90;
-- VALUES 없이 SELECT 결과를 통째로 삽입
```

## 에러 나는 상황

```sql
-- SAMPLE: COL1(PK), COL2, COL3
INSERT INTO SAMPLE (COL2, COL3) VALUES ('A', 'B');
-- COL1 지정 안 함 → NULL 삽입 시도 → PK 위반 → 에러!
```

```
DEFAULT 동작:
  컬럼 아예 생략    → DEFAULT 값 자동 삽입
  VALUES 에 명시   → DEFAULT 무시, 명시한 값 삽입
```

---

---

# ③ UPDATE — 데이터 수정

```
🚨 WHERE 없으면 모든 행이 바뀜
```

```sql
UPDATE 사원
SET
    부서명 = 'AI팀',
    연봉   = 5000
WHERE 사번 = 101;
```

---

---

# ④ DELETE — 데이터 삭제

```
🚨 WHERE 없으면 테이블이 텅 빔
```

```sql
DELETE FROM 사원
WHERE 사번 = 102;
-- 행(Row) 전체 삭제 / 컬럼명 안 씀
```

---

---

# ⑤ ON CONFLICT — PostgreSQL UPSERT ⭐️

```
INSERT 를 시도하다가 PK/UNIQUE 충돌 나면 어떻게 할지 지정
Oracle MERGE 보다 간결
```

## DO NOTHING — 충돌 시 그냥 무시

```sql
INSERT INTO logs (id, msg) VALUES (1, 'hello')
ON CONFLICT (id) DO NOTHING;
-- id=1 이 이미 있으면 아무것도 안 함 (에러 없음)
-- 중복 삽입 방지 / 멱등성 보장할 때
```

## DO UPDATE SET — 충돌 시 갱신 (UPSERT)

```sql
INSERT INTO er_hospitals (hpid, hpname, region)
VALUES ('A001', '신_병원명', '서울')
ON CONFLICT (hpid) DO UPDATE SET
    hpname = EXCLUDED.hpname,   -- 새 값으로 갱신
    region = EXCLUDED.region,
    updated_at = NOW();          -- 갱신 시각 기록
```

## EXCLUDED 가 뭔가

```
EXCLUDED = "지금 INSERT 하려다 충돌난 새 데이터" 가상 테이블

기존 DB: hpid='A001', hpname='구_병원명'
새 데이터: hpid='A001', hpname='신_병원명'

ON CONFLICT DO UPDATE SET hpname = EXCLUDED.hpname
→ EXCLUDED.hpname = '신_병원명' (새로 들어온 값)
→ 결과: hpname = '신_병원명' 으로 갱신
```

```sql
-- 일부 컬럼만 갱신 / 일부는 기존 유지
ON CONFLICT (hpid) DO UPDATE SET
    hpname     = EXCLUDED.hpname,       -- 새 값으로 갱신
    updated_at = NOW(),                  -- 갱신 시각
    created_at = er_hospitals.created_at -- 최초 등록일 유지 (기존 값)
```

```
EXCLUDED.컬럼    → 새로 들어온 값 (갱신할 때)
테이블명.컬럼    → 기존 테이블 값 (유지할 때)
NOW()           → DB 서버 현재 시각
```

## DO NOTHING vs DO UPDATE 비교

|항목|DO NOTHING|DO UPDATE SET|
|---|---|---|
|충돌 시|무시|기존 행 갱신|
|적합한 상황|중복 방지 / 로그|최신 데이터 유지|
|EXCLUDED|불필요|새 값 참조|

>[[Airflow_Hooks#ON CONFLICT + EXCLUDED — UPSERT 핵심 ️]] 참고 
---

---

# ⑥ MERGE — Oracle UPSERT

```
Oracle 에서 UPSERT 하는 방법
소스 테이블 → 타겟 테이블 로 데이터를 병합
```

```sql
MERGE INTO 타겟테이블 T
USING 소스테이블 S
ON (T.사번 = S.사번)         -- 기준 컬럼 비교

WHEN MATCHED THEN            -- 일치하는 행 있을 때 → UPDATE
    UPDATE SET
        T.부서명 = S.부서명,
        T.연봉   = S.연봉

WHEN NOT MATCHED THEN        -- 일치하는 행 없을 때 → INSERT
    INSERT (사번, 이름, 부서명)
    VALUES (S.사번, S.이름, S.부서명);
```

```
WHEN MATCHED 에 조건 추가 가능:
  WHEN MATCHED THEN
      UPDATE SET T.연봉 = S.연봉
      WHERE S.실적 > 90   ← 매칭됐어도 조건 만족 시만 갱신
```

```sql
-- ❌ WHEN NOT MATCHED 안에 SELECT 금지
WHEN NOT MATCHED THEN
    INSERT (사번) SELECT S.사번 FROM ...  -- 에러!

-- ✅ VALUES 만 가능
WHEN NOT MATCHED THEN
    INSERT (사번) VALUES (S.사번)
-- USING 에서 이미 SELECT 로 가져왔으므로 안에서 또 SELECT 불가
```

---

---

# Oracle vs PostgreSQL UPSERT 비교

|구분|Oracle MERGE INTO|PostgreSQL ON CONFLICT|
|---|---|---|
|방식|소스·타겟 명시 분리|INSERT 후 충돌 처리|
|없을 때|WHEN NOT MATCHED → INSERT|기본 INSERT|
|있을 때|WHEN MATCHED → UPDATE|ON CONFLICT DO UPDATE|
|새 데이터 참조|`S.컬럼명`|`EXCLUDED.컬럼명`|
|코드 길이|길다|짧다|

---

---

# Python 에서 INSERT

```python
# 단건 INSERT
cursor.execute("""
    INSERT INTO user_logs (user_name, price, timestamp)
    VALUES (%s, %s, NOW())
""", (data["user_name"], data["price"]))
# NOW() 는 SQL 안에 쓰니까 파이썬 튜플에서 빠짐

# 대량 INSERT — execute_values (훨씬 빠름)
from psycopg2.extras import execute_values

sql = "INSERT INTO er_hospitals (hpid, hpname) VALUES %s"
execute_values(cur, sql, [("A001", "서울대"), ("A002", "세브란스")])
```

> [[Python_Database_Connect]] / [[Airflow_Hooks]] 참고