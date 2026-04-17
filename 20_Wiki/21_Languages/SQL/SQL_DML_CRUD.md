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

## DELETE + JOIN / USING — 다른 테이블 참조해서 삭제 ⭐️

```
같은 테이블 / 다른 테이블의 값을 기준으로
조건에 맞는 행만 골라서 삭제

MySQL 과 PostgreSQL 문법이 다름
```

## MySQL — DELETE + JOIN

```sql
-- 중복 이메일 제거 (id 가 큰 것 삭제)
DELETE p1
FROM Person p1
JOIN Person p2
ON p1.email = p2.email
AND p1.id > p2.id;
```

```
DELETE 뒤에 별칭을 써서 어떤 테이블에서 삭제할지 지정
p1 = 삭제 대상 / p2 = 비교 기준
같은 email 중 id 가 큰 것(나중에 들어온 중복) 삭제
```

## PostgreSQL — DELETE + USING

```sql
-- 중복 이메일 제거 (id 가 큰 것 삭제)
DELETE FROM Person p1
USING Person p2
WHERE p1.email = p2.email
AND p1.id > p2.id;
```

```
PostgreSQL 은 JOIN 대신 USING 키워드 사용
DELETE FROM 삭제테이블
USING 참조테이블
WHERE 조건

Self JOIN 패턴:
  같은 테이블을 두 번 참조
  p1 = 삭제 대상 / p2 = 비교 기준
```

## MySQL vs PostgreSQL 비교

```sql
-- MySQL
DELETE p1
FROM Person p1
JOIN Person p2 ON p1.email = p2.email AND p1.id > p2.id;

-- PostgreSQL
DELETE FROM Person p1
USING Person p2
WHERE p1.email = p2.email AND p1.id > p2.id;
```

|구분|MySQL|PostgreSQL|
|---|---|---|
|문법|`DELETE 별칭 FROM ... JOIN`|`DELETE FROM ... USING`|
|참조 방법|JOIN|USING|
|삭제 대상 지정|DELETE 뒤 별칭|DELETE FROM 테이블|

## 실전 패턴 — 중복 제거

```sql
-- 같은 email 중 id 가 작은 것만 남기기
-- (id 가 큰 중복 제거 → 가장 먼저 등록된 것 유지)

-- PostgreSQL
DELETE FROM Person p1
USING Person p2
WHERE p1.email = p2.email
AND p1.id > p2.id;

-- 결과 확인
SELECT * FROM Person;
-- 각 email 당 id 가 가장 작은 행만 남음
```

```
문제 키워드:
  "중복 제거" / "이메일 중복" / "최초 등록만 유지"
  → DELETE + Self JOIN (MySQL) / DELETE + USING (PostgreSQL)
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


# ⑦ VALUES %s — execute_values 대량 INSERT 패턴 ⭐️

## %s 가 뭔가

```
psycopg2 의 플레이스홀더
파이썬 값을 SQL 에 안전하게 넣어주는 자리

%s 하나 = 값 하나
VALUES %s = "여러 행의 묶음 전체" (execute_values 전용)
```

```python
# 단건 INSERT — %s 하나씩 나열
cur.execute(
    "INSERT INTO raw_exhibitions (exhibition_id, title) VALUES (%s, %s)",
    ("26002594", "페르난도 보테로展")
)

# 대량 INSERT — VALUES %s (execute_values 전용)
from psycopg2.extras import execute_values

sql = "INSERT INTO raw_exhibitions (exhibition_id, title) VALUES %s"
execute_values(cur, sql, [
    ("26002594", "페르난도 보테로展"),
    ("26003427", "맥스 시덴토프 개인전"),
    ("26003180", "인상주의를 넘어"),
])
```

```
execute_values 가 하는 일:
  VALUES %s → VALUES (%s,%s), (%s,%s), (%s,%s) 로 자동 확장
  → SQL 1번으로 N건 삽입 (N번 왕복 없음)

단건 execute() 반복 vs execute_values:
  execute() 반복  → 500건 = 500번 DB 왕복 (느림)
  execute_values  → 500건 = 1번 DB 왕복   (빠름) ✅
```

## 실전 패턴 — 컬럼 많을 때

```python
sql = """
INSERT INTO raw_exhibitions (
    exhibition_id, title, subtitle,
    venue, location, address,
    start_date, end_date, hours,
    prices_raw, age_limit,
    category, genre,
    day_rank, week_rank, month_rank, rank,
    image_url, detail_url,
    notice,
    is_active, crawled_at, updated_at
) VALUES %s
"""

values = [
    (
        ex["exhibition_id"],   # exhibition_id
        ex.get("title"),       # title
        ex.get("subtitle"),    # subtitle
        ex.get("venue"),       # venue
        ex.get("location"),    # location
        ex.get("address"),     # address
        ex.get("start_date"),  # start_date
        ex.get("end_date"),    # end_date
        ex.get("hours"),       # hours
        to_jsonb(ex.get("prices_raw")),  # prices_raw (JSONB)
        ex.get("age_limit"),   # age_limit
        ex.get("category"),    # category
        ex.get("genre"),       # genre
        ex.get("day_rank"),    # day_rank
        ex.get("week_rank"),   # week_rank
        ex.get("month_rank"),  # month_rank
        ex.get("rank"),        # rank
        ex.get("image_url"),   # image_url
        ex.get("detail_url"),  # detail_url
        ex.get("notice"),      # notice
        ex.get("is_active", True),     # is_active
        ex.get("crawled_at", now),     # crawled_at
        now,                           # updated_at
    )
    for ex in exhibitions
]

execute_values(cur, sql, values)
```

```
컬럼 순서 = 튜플 순서 일치 필수 ⚠️
컬럼이 23개면 튜플 원소도 23개여야 함
순서 틀리면 에러 없이 잘못된 컬럼에 저장될 수 있음
→ SQL 의 컬럼 목록과 values 튜플을 나란히 놓고 맞춰가며 작성
```

---

---

# ⑧ ON CONFLICT 완전 정복 ⭐️

## 왜 필요한가

```
크롤러가 매일 실행됨
같은 exhibition_id 데이터가 또 들어올 수 있음
→ INSERT 시도 → PK 충돌 → 에러

ON CONFLICT:
  충돌 나면 어떻게 할지 미리 지정
  에러 없이 처리
```

## 기본 구조

```sql
INSERT INTO 테이블 (컬럼1, 컬럼2, ...)
VALUES (값1, 값2, ...)
ON CONFLICT (충돌_기준_컬럼) DO 처리방식;
--           ↑
--    UNIQUE 또는 PK 컬럼 지정
```

## DO NOTHING vs DO UPDATE 비교

```sql
-- DO NOTHING: 충돌 시 그냥 넘어감
INSERT INTO logs (id, msg) VALUES (1, 'hello')
ON CONFLICT (id) DO NOTHING;
-- id=1 이미 존재 → 아무 일도 일어나지 않음 (에러도 없음)

-- DO UPDATE: 충돌 시 기존 행 갱신 (UPSERT)
INSERT INTO logs (id, msg) VALUES (1, 'new_hello')
ON CONFLICT (id) DO UPDATE SET
    msg = EXCLUDED.msg;
-- id=1 이미 존재 → msg 를 'new_hello' 로 갱신
```

## EXCLUDED — 핵심 개념

```
EXCLUDED = 지금 INSERT 하려다 충돌난 새 데이터 가상 테이블

예시:
  DB에 기존 데이터: exhibition_id='26002594', title='구 제목', week_rank=5
  새로 들어온 데이터: exhibition_id='26002594', title='새 제목', week_rank=3

  ON CONFLICT (exhibition_id) DO UPDATE SET
      title     = EXCLUDED.title,     -- '새 제목' (새로 들어온 값)
      week_rank = EXCLUDED.week_rank  -- 3 (새로 들어온 값)

  결과: title='새 제목', week_rank=3 으로 갱신
```

```
EXCLUDED.컬럼  → 새로 들어온 값 (갱신하고 싶을 때)
테이블명.컬럼  → 기존에 있던 값 (유지하고 싶을 때)
```

## 갱신할 컬럼 / 유지할 컬럼 분리

```sql
ON CONFLICT (exhibition_id) DO UPDATE SET
    title      = EXCLUDED.title,               -- ✅ 새 값으로 갱신
    week_rank  = EXCLUDED.week_rank,           -- ✅ 새 값으로 갱신
    updated_at = NOW(),                         -- ✅ 갱신 시각 기록
    crawled_at = raw_exhibitions.crawled_at    -- ✅ 최초 크롤링 시각 유지 (기존값)
    --           ↑ 테이블명.컬럼 = 기존 값 유지
```

```
crawled_at 은 처음 수집된 시각을 보존하고 싶음
→ EXCLUDED.crawled_at (새 값) 대신
  raw_exhibitions.crawled_at (기존 값) 사용
```

## 실전 전체 패턴 — exhibition 프로젝트

```sql
INSERT INTO raw_exhibitions (
    exhibition_id, title, subtitle,
    venue, location, address,
    start_date, end_date, hours,
    prices_raw, age_limit,
    category, genre,
    day_rank, week_rank, month_rank, rank,
    image_url, detail_url,
    notice,
    is_active, crawled_at, updated_at
) VALUES %s
ON CONFLICT (exhibition_id) DO UPDATE SET
    title       = EXCLUDED.title,
    subtitle    = EXCLUDED.subtitle,
    venue       = EXCLUDED.venue,
    location    = EXCLUDED.location,
    address     = EXCLUDED.address,
    start_date  = EXCLUDED.start_date,
    end_date    = EXCLUDED.end_date,
    hours       = EXCLUDED.hours,
    prices_raw  = EXCLUDED.prices_raw,
    age_limit   = EXCLUDED.age_limit,
    category    = EXCLUDED.category,
    genre       = EXCLUDED.genre,
    day_rank    = EXCLUDED.day_rank,
    week_rank   = EXCLUDED.week_rank,
    month_rank  = EXCLUDED.month_rank,
    rank        = EXCLUDED.rank,
    image_url   = EXCLUDED.image_url,
    detail_url  = EXCLUDED.detail_url,
    notice      = EXCLUDED.notice,
    is_active   = EXCLUDED.is_active,
    updated_at  = EXCLUDED.updated_at
    -- crawled_at 은 DO UPDATE 목록에 없음 → 기존 값 자동 유지
```

```
crawled_at 을 DO UPDATE SET 목록에서 빼면
→ 충돌 시 crawled_at 은 건드리지 않음
→ 최초 크롤링 시각이 그대로 보존됨
```

## 충돌 기준 컬럼 — 여러 개도 가능

```sql
-- 단일 컬럼 충돌 기준
ON CONFLICT (exhibition_id) DO UPDATE SET ...

-- 복합 컬럼 충돌 기준 (UNIQUE 제약이 두 컬럼 조합에 걸려있을 때)
ON CONFLICT (exhibition_id, snapshot_date) DO UPDATE SET ...
-- raw_exhibition_history: UNIQUE (exhibition_id, snapshot_date)
-- 같은 전시, 같은 날짜로 또 들어오면 갱신
```

## DO UPDATE 에 조건 추가 — WHERE

```sql
-- 새 week_rank 가 더 낮을 때만 갱신 (순위가 올랐을 때만 업데이트)
ON CONFLICT (exhibition_id) DO UPDATE SET
    week_rank = EXCLUDED.week_rank
WHERE EXCLUDED.week_rank < raw_exhibitions.week_rank;
--    ↑ 새 순위           ↑ 기존 순위
```

## ON CONFLICT 실행 결과 확인

```python
conn = None
try:
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor() as cur:
        execute_values(cur, sql, values)
        # rowcount: 실제로 처리된 행 수
        # INSERT = 새로 들어간 것 + UPDATE 된 것 합산
        print(f"처리된 행: {cur.rowcount}개")
    conn.commit()
finally:
    if conn:
        conn.close()
```

## ON CONFLICT — SQLD 시험 관련

```
SQLD 시험은 Oracle 기준 → ON CONFLICT 없음
Oracle 에서 같은 기능 = MERGE INTO

ON CONFLICT  → PostgreSQL
MERGE INTO   → Oracle
INSERT OR REPLACE → MySQL / SQLite
```

|기능|PostgreSQL|Oracle|
|---|---|---|
|충돌 무시|`ON CONFLICT DO NOTHING`|`MERGE ... WHEN NOT MATCHED`|
|충돌 시 갱신|`ON CONFLICT DO UPDATE SET`|`MERGE ... WHEN MATCHED`|
|새 데이터 참조|`EXCLUDED.컬럼`|`S.컬럼` (소스 테이블)|






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

## values 리스트 만들기 — for ex in exhibitions

```python
def upsert_exhibitions(self, exhibitions: list[dict[str, Any]]) -> int:
    #                                     ↑
    #                              딕셔너리들의 리스트
```

```
exhibitions = [
    {"exhibition_id": "26002594", "title": "페르난도 보테로展"},
    {"exhibition_id": "26003427", "title": "맥스 시덴토프 개인전"},
]

for ex in exhibitions:
    # ex = 딕셔너리 1개
    # ex["exhibition_id"] → "26002594"
    # ex.get("title")     → "페르난도 보테로展"
```

```python
# 리스트 컴프리헨션으로 튜플 묶음 생성
values = [
    (
        ex["exhibition_id"],        # 반드시 있어야 해서 []
        ex.get("title"),            # 없으면 None → DB 에 NULL
        ex.get("is_active", True),  # 없으면 기본값 True
    )
    for ex in exhibitions           # list 를 순회하며 dict 하나씩
]
# 결과:
# [
#   ("26002594", "페르난도 보테로展", True),
#   ("26003427", "맥스 시덴토프 개인전", True),
# ]

execute_values(cur, sql, values)
```