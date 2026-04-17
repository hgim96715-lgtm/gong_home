---
aliases:
  - PostgreSQL 연결
  - DataGrip 연결
  - init.sql
  - psql
tags:
  - SQL
  - PostgreSQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[Docker_Volumes]]"
  - "[[SQL_DDL_Create]]"
  - "[[Spark_JDBC_PostgreSQL]]"
---
# 🐘 PostgreSQL_Setup

## 한 줄 요약

```
PostgreSQL 을 Docker 로 띄우고
DataGrip 으로 연결해서 바로 쓰는 환경 세팅
```

---

---

# ① init.sql — 테이블 자동 생성

```
docker-entrypoint-initdb.d/ 안에 마운트된 .sql 파일은
컨테이너 최초 실행 시 딱 한 번 자동 실행됨
```

```yaml
# docker-compose.yml 마운트 설정
volumes:
  - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```

```
실행 순서:
  docker compose up -d
  → PostgreSQL 컨테이너 시작
  → 볼륨이 비어있으면 init.sql 자동 실행
  → 테이블 생성 완료
```

```
⚠️ init.sql 이 실행 안 될 때:
  이미 볼륨이 존재하면 init.sql 을 다시 실행하지 않음

해결:
  docker compose down -v   ← 볼륨까지 삭제
  docker compose up -d     ← 다시 올리면 init.sql 실행됨
```

```
💡 init.sql 에 테이블 추가했는데 반영 안 될 때:
  파일만 수정해도 볼륨이 이미 있으면 실행 안 됨

순서:
  docker compose down -v   ← 기존 볼륨(데이터) 전부 삭제
  (init.sql 수정)
  docker compose up -d     ← 재시작 → init.sql 다시 실행
  DataGrip F5 새로고침     ← 새 테이블 확인
```

> ⚠️ `down -v` 는 기존 데이터도 전부 삭제 운영 중인 DB 에서는 절대 사용 금지 개발 초기 스키마 세팅 단계에서만 사용

> `down -v` 없이 수정 → ⑥ 스키마 수정 참고

---

---

# ② 자주 쓰는 자료형

|자료형|설명|예시|
|---|---|---|
|`SERIAL`|자동 증가 정수 (PK 용)|1, 2, 3 ...|
|`INTEGER`|정수 (-2억 ~ 2억)|100, 200|
|`BIGINT`|큰 정수 (-900경 ~ 900경)|타임스탬프(ms), 대용량 ID|
|`VARCHAR(n)`|최대 n 자리 문자열|`VARCHAR(20)`|
|`TEXT`|길이 제한 없는 문자열|긴 설명, 메시지|
|`NUMERIC(p, s)`|소수점 포함 정밀 숫자|금액, 좌표|
|`TIMESTAMP`|날짜 + 시각|`2026-03-05 21:00:00`|
|`DATE`|날짜만|`2026-03-05`|
|`BOOLEAN`|참/거짓|`TRUE`, `FALSE`|
|`ENUM`|미리 정의한 고정 값 목록|`'정시'`, `'지연'`|

## NUMERIC(p, s) — p 와 s 가 뭔가? ⭐️

```
NUMERIC(p, s)
  p = precision  전체 자릿수 (소수점 포함 총 자릿수)
  s = scale      소수점 이하 자릿수

예시:
  NUMERIC(5, 2)
  → 전체 5자리, 소수점 이하 2자리
  → 저장 가능 범위: -999.99 ~ 999.99
  → 123.45  ✅
  → 1234.5  ❌ (정수부가 4자리 → 초과)
  → 12.345  ❌ (소수부가 3자리 → 초과)

  NUMERIC(10, 7)  ← 위경도에 사용
  → 전체 10자리, 소수점 이하 7자리
  → 127.0851566  ✅  (정수부 3 + 소수부 7 = 10)
  → 37.4881325   ✅
```

## INTEGER vs BIGINT

```
INTEGER   4바이트  약 -21억 ~ 21억
BIGINT    8바이트  약 -900경 ~ 900경

BIGINT 쓰는 경우:
  Unix 타임스탬프(밀리초)  → 1710000000000  (13자리, INTEGER 초과)
  Kafka 오프셋            → 매우 큰 수
  대용량 데이터 ID         → INTEGER 범위 초과 가능성

  일반 병상 수 / 카운트   → INTEGER 로 충분
```

## ENUM — 고정 값 목록

```
미리 정해진 값만 저장 가능
입력값 자동 검증 + 저장 공간 효율적

주의:
  CREATE TYPE 을 CREATE TABLE 보다 먼저 선언
  값이 자주 바뀌면 VARCHAR 권장
```

```sql
-- ENUM 타입 생성 (CREATE TABLE 앞에 위치해야 함)
CREATE TYPE delay_status AS ENUM ('정시', '소폭지연', '지연', '대폭지연');

CREATE TABLE train_delay (
    id         SERIAL PRIMARY KEY,
    dep_status delay_status,
    arr_status delay_status
);

-- 값 추가
ALTER TYPE delay_status ADD VALUE '운행취소';

-- ENUM 목록 확인
SELECT enum_range(NULL::delay_status);
```

## CHECK — 값 범위 제약

```
INSERT / UPDATE 시 조건 검사
위반하면 에러 → 잘못된 데이터 입력 자체를 차단
```

```sql
CREATE TABLE er_realtime (
    id         SERIAL PRIMARY KEY,
    hvec       INTEGER CHECK (hvec >= 0),                    -- 음수 불가
    hvctayn    VARCHAR(10),   -- Y/N (1 초과 값 올 수 있어서 10으로 여유)
    saturation NUMERIC(5,2) CHECK (saturation BETWEEN 0 AND 100)  -- 0~100
);

-- 제약 이름 붙이기 (에러 메시지에 표시됨)
ALTER TABLE er_realtime
  ADD CONSTRAINT beds_positive CHECK (hvec >= 0);
```

```
CHECK vs ENUM:
  ENUM   정해진 문자열 목록 중 하나 (타입 정의 필요)
  CHECK  더 복잡한 조건 (범위 / IN / BETWEEN)
         타입 정의 없이 바로 사용 가능

  값 종류가 적으면 CHECK 가 더 편함
  VARCHAR(1) CHECK (col IN ('Y','N')) ← ENUM 없이 간단히
```

```sql
-- 자료형 사용 예시
id           SERIAL PRIMARY KEY        -- 자동 증가 PK
hpid         VARCHAR(20)               -- 병원 ID 문자열
hvec         INTEGER                   -- 응급실 가용병상 (정수)
notice_msg  TEXT                      -- 긴 메시지
wgs84_lat    NUMERIC(10, 7)            -- 위도 (소수 7자리)
wgs84_lon    NUMERIC(10, 7)            -- 경도 (소수 7자리)
avg_beds     NUMERIC(5, 2)             -- 평균 병상 (123.45)
saturation   NUMERIC(5, 2)             -- 포화율 (99.99%)
created_at   TIMESTAMP DEFAULT NOW()   -- 저장 시각 자동
```

---

---

# ③ DataGrip 연결

```
docker compose up -d 후 DataGrip 에서 연결

DataGrip → New → Data Source → PostgreSQL

Host:      localhost
Port:      5433        ← docker-compose.yml 의 호스트 포트
Database:  <DB명>
User:      <유저명>
Password:  <패스워드>
```

```
연결 후 확인 경로:
  <DB명> → public → Tables
    ├── 테이블1
    └── 테이블2
```

> 처음 연결 시 드라이버 다운로드 팝업 → Download 클릭

## ⚠️ "user does not exist" 에러

```
원인: 로컬 5432 포트가 이미 사용 중
     → 컨테이너 포트포워딩 실패

확인:
  lsof -i :5432   ← 뭔가 뜨면 로컬 PostgreSQL 이 점유 중

해결:
  docker-compose.yml 포트를 5433:5432 로 변경
  DataGrip 도 Port: 5433 으로 수정

  컨테이너 내부는 여전히 5432
  → 컨테이너끼리 통신할 때는 5432 그대로 사용
```

> [[Linux_Port#② lsof — 포트 점유 확인]] 참조

---

---

# ④ psql — PostgreSQL 전용 CLI 클라이언트

```
psql = PostgreSQL 에 접속해서 SQL 을 직접 실행하는 명령줄 도구

DataGrip  GUI (마우스로 클릭)
psql      CLI (터미널에서 타이핑)

언제 psql 쓰나:
  DataGrip 없는 서버 환경
  빠르게 쿼리 한 줄 확인할 때
  docker exec 로 컨테이너 안에서 바로 접속할 때
  스크립트에서 자동화할 때
```

```bash
# 컨테이너 내부 psql 접속
docker exec -it <container_name> psql -U <유저명> -d <DB명>

# 예시 (exhibition 프로젝트)
docker exec -it exhibition-postgres psql -U postgres -d exhibition_db
#                 ↑ 컨테이너 이름              ↑ DB 유저        ↑ DB 이름
#                 (Docker)                  (.env)          (.env)
```

```
⚠️ container_name ≠ DB명
  exhibition-postgres  → Docker 컨테이너 이름 (docker-compose.yml 의 container_name)
  exhibition_db        → PostgreSQL 안의 데이터베이스 이름 (.env 의 POSTGRES_DB)
```

```bash
# 접속 성공 시 프롬프트
exhibition_db=#    ← 여기서 SQL 또는 메타 명령어 입력
```

---

## psql 메타 명령어 — \로 시작하는 단축 명령어

```
SQL 문법이 아님
psql 클라이언트가 직접 처리하는 명령어
세미콜론(;) 안 붙여도 됨
백슬래시(\)로 시작
```

### \d 계열 — 구조 확인

|명령어|뭘 보여주나|
|---|---|
|`\dt`|테이블 목록|
|`\dt+`|테이블 목록 + 크기 + 설명|
|`\d 테이블명`|컬럼명 / 타입 / 제약조건|
|`\d+ 테이블명`|`\d` + 인덱스 / 크기 / COMMENT ON 내용|
|`\di`|인덱스 목록|
|`\di+`|인덱스 목록 + 크기|
|`\dv`|뷰 목록|
|`\dn`|스키마 목록|
|`\ds`|시퀀스 목록 (SERIAL 컬럼이 만드는 것)|

```
+ 를 붙이면 더 상세한 정보가 추가되는 패턴
\d → \d+  /  \dt → \dt+  /  \di → \di+  모두 동일한 방식
```

### \d vs \d+ 차이 — COMMENT ON 확인

```bash
\d raw_exhibitions
```

```
 Column        | Type         | Nullable | Default
---------------+--------------+----------+--------
 id            | integer      | not null |
 exhibition_id | varchar(50)  | not null |
 title         | varchar(500) | not null |
 prices_raw    | jsonb        |          |
```

```bash
\d+ raw_exhibitions
```

```
 Column        | Type         | Nullable | Default | Description
---------------+--------------+----------+---------+-------------------------------------
 id            | integer      | not null |         |
 exhibition_id | varchar(50)  | not null |         | goodsCode — 인터파크 전시 고유 ID
 title         | varchar(500) | not null |         |
 prices_raw    | jsonb        |          |         | /v1/goods/{goodsCode}/prices/group
                                                              ↑
                                              COMMENT ON 으로 저장한 내용이 여기 표시됨
Indexes:
    "idx_ex_location" btree (location)
    "idx_ex_active_period" btree (is_active, start_date, end_date)
    "idx_ex_week_rank" btree (week_rank) WHERE week_rank IS NOT NULL
```

```
\d+  를 쓰면 한 번에 확인 가능한 것:
  컬럼 구조
  인덱스 목록
  COMMENT ON 내용 (Description 컬럼)
  테이블 크기
```

### DB / 유저 / 권한 확인

|명령어|뭘 보여주나|
|---|---|
|`\l`|DB 목록 + 소유자 + 권한|
|`\du`|유저 목록 + 역할 + 슈퍼유저 여부|
|`\dp 테이블명`|특정 테이블 권한 목록|
|`\conninfo`|현재 접속 정보 (DB명 / 유저 / 포트)|
|`\c DB명`|다른 DB 로 전환|

### 기타

|명령어|동작|
|---|---|
|`\q`|psql 종료|
|`\i 파일경로`|외부 SQL 파일 실행|
|`\e`|외부 편집기 열기 (긴 쿼리 작성할 때)|
|`\timing`|쿼리 실행 시간 표시 ON/OFF|
|`\x`|결과를 세로로 표시 (컬럼 많을 때 유용)|

```bash
# \x 예시 — 컬럼이 많아서 가로로 안 보일 때
\x
SELECT * FROM raw_exhibitions LIMIT 1;

# 출력:
-[ RECORD 1 ]--+-----------------------------
id             | 1
exhibition_id  | 26002594
title          | ［얼리버드 40%］ 페르난도 보테로展
venue          | 예술의전당 한가람디자인미술관
...
```

---

## 자주 쓰는 실전 패턴

```sql
-- 테이블 목록 확인
\dt

-- 특정 테이블 구조 + 인덱스 + COMMENT 한 번에
\d+ raw_exhibitions

-- 인덱스 제대로 생성됐는지 확인
\di

-- 현재 어떤 DB 에 접속 중인지 확인
\conninfo

-- 데이터 몇 건인지 빠르게 확인
SELECT COUNT(*) FROM raw_exhibitions;

-- 최근 적재된 데이터 확인
SELECT exhibition_id, title, crawled_at
FROM raw_exhibitions
ORDER BY crawled_at DESC
LIMIT 5;

-- 접속 종료
\q
```

> [[Docker_Container_Interaction]] , [[Docker_Compose_Commands]] 참조


---

---

# ⑤ DB · USER · 권한 관리

```
새 프로젝트 시작 흐름:
  CREATE DATABASE  → DB 만들기
  CREATE USER      → 전용 유저 만들기
  GRANT            → 권한 주기
  ALTER DATABASE   → 소유권 설정
```

## DB 생성 / 삭제

```sql
CREATE DATABASE mydb;
DROP DATABASE mydb;    -- 접속 중인 유저 없어야 함
\l                     -- DB 목록 확인
```

## USER 생성 / 삭제

```sql
CREATE USER myuser WITH PASSWORD 'mypassword';
DROP USER myuser;
ALTER USER myuser WITH PASSWORD 'newpassword';
\du                    -- 유저 목록 확인
```

## GRANT — 권한 부여

```sql
-- DB 전체 권한
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

-- 특정 테이블만
GRANT SELECT, INSERT ON TABLE mytable TO myuser;

-- 모든 테이블
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO myuser;

-- 소유권 이전
ALTER DATABASE mydb OWNER TO myuser;

-- 권한 회수
REVOKE ALL PRIVILEGES ON DATABASE mydb FROM myuser;
```

## 실전 패턴 — 프로젝트 전용 세팅

```sql
-- init.sql 에 추가하는 패턴
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```

```
각 명령어 역할:
  CREATE DATABASE   DB 생성
  CREATE USER       전용 유저 생성
  GRANT ALL         DB 접근 + 테이블 생성 등 전체 권한
  ALTER OWNER       소유권까지 넘김 → 완전한 독립 환경
```

## 확인 명령어

```sql
\l          -- DB 목록 + 소유자 + 권한
\du         -- 유저 목록 + 역할 + 슈퍼유저 여부
\dp         -- 테이블 권한 목록
\conninfo   -- 현재 접속 정보
\c mydb     -- 특정 DB 로 전환
\dn         -- 스키마 목록
```

## 스키마 — public 이 뭔가?

```
스키마 = DB 안에 있는 폴더 (테이블을 묶는 네임스페이스)

PostgreSQL 기본 스키마: public
→ 별도 지정 없으면 모든 테이블이 public 에 생성
→ Superset Dataset 등록 시 "스키마 = public" 선택하는 이유

명시적으로 접근:
  SELECT * FROM public.train_realtime;
```

---

---

# ⑥ 스키마 수정 — down -v 없이 변경하기 ⭐️

```
down -v 하면:
  PostgreSQL 데이터 삭제
  Superset 차트 / 대시보드 / DB 연결 전부 삭제
  → 절대 일상적으로 쓰면 안 됨

실제 테이블 구조 수정 시:
  psql 직접 접속 → ALTER TABLE 로 처리
```

## ALTER TABLE — 구조 변경

```sql
-- 컬럼 추가
ALTER TABLE er_realtime ADD COLUMN new_col VARCHAR(50);

-- 컬럼 삭제
ALTER TABLE er_realtime DROP COLUMN old_col;

-- 타입 변경
ALTER TABLE er_realtime ALTER COLUMN status TYPE TEXT;

-- 컬럼 이름 변경
ALTER TABLE er_realtime RENAME COLUMN old_name TO new_name;

-- 기본값 설정
ALTER TABLE er_realtime ALTER COLUMN data_type SET DEFAULT 'er_realtime';

-- NOT NULL 추가 / 제거
ALTER TABLE er_realtime ALTER COLUMN hpid SET NOT NULL;
ALTER TABLE er_realtime ALTER COLUMN hpid DROP NOT NULL;
```

## 데이터 수정 / 삭제

```sql
-- 조건부 수정
UPDATE er_realtime SET region = '서울' WHERE region IS NULL;

-- 전체 데이터 삭제 (구조 유지)
TRUNCATE TABLE er_realtime;

-- 오래된 데이터 삭제
DELETE FROM er_realtime WHERE created_at < NOW() - INTERVAL '7 days';
```

## ⭐️ psql 수정 후 init.sql 도 반드시 수정

```
psql 로 직접 수정하면 현재 DB 는 즉시 반영됨
BUT init.sql 은 그대로
→ 나중에 down -v 하면 수정 전 상태로 돌아감

수정 작업 순서:
  1. psql 접속 → ALTER TABLE 실행 (즉시 반영)
  2. init.sql 도 동일하게 수정 (미래 재구성 대비)
  3. DataGrip F5 새로고침
  4. Superset → Dataset → Edit → Sync columns from source
```

## down -v 가 필요한 경우 vs 불필요한 경우

```
down -v 필요:
  CREATE TYPE / ENUM 추가 (타입은 ALTER 로 추가 불가)
  DB 자체를 새로 만들어야 할 때
  볼륨이 꼬여서 컨테이너가 아예 안 올라올 때

down -v 불필요 (psql 직접):
  컬럼 추가 / 삭제 / 타입 변경  → ALTER TABLE
  데이터 수정 / 삭제            → UPDATE / DELETE / TRUNCATE
  인덱스 추가                   → CREATE INDEX
  새 테이블 추가                → CREATE TABLE (psql 에서 바로 실행)
```

---

---

# ⑦ INDEX — 검색 속도 높이기

## 한 줄 요약

> **"책의 목차. 특정 컬럼에 미리 정렬된 주소록을 만들어두어 조회 속도를 높이는 구조."**

---

## 왜 필요한가?

```
인덱스 없을 때 → Full Table Scan
테이블 전체를 처음부터 끝까지 읽음
데이터 10만 건이면 10만 번 비교

인덱스 있을 때 → Index Scan
미리 정렬된 주소록에서 위치만 찾아감
```

```sql
-- 인덱스 없는 상태
SELECT * FROM raw_exhibitions WHERE location = '서울';
-- → 전체 스캔 (느림)

-- 인덱스 생성 후
CREATE INDEX idx_ex_location ON raw_exhibitions (location);
SELECT * FROM raw_exhibitions WHERE location = '서울';
-- → 인덱스 통해 바로 접근 (빠름)
```

---

## 기본 문법

```sql
-- 단일 컬럼 인덱스
CREATE INDEX 인덱스명 ON 테이블명 (컬럼명);

-- 복합 인덱스 (여러 컬럼 동시 조건)
CREATE INDEX 인덱스명 ON 테이블명 (컬럼1, 컬럼2, 컬럼3);

-- Partial Index (조건 있는 부분 인덱스)
CREATE INDEX 인덱스명 ON 테이블명 (컬럼명)
WHERE 조건;

-- 인덱스 삭제
DROP INDEX 인덱스명;

-- 인덱스 확인
\di 테이블명   -- psql
-- 또는
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'raw_exhibitions';
```

---

## 복합 인덱스 — 컬럼 순서가 중요하다

```sql
CREATE INDEX idx_ex_active_period
    ON raw_exhibitions (is_active, start_date, end_date);
```

```
이 인덱스가 동작하는 쿼리:
  WHERE is_active = TRUE                          ✅ (첫 번째 컬럼)
  WHERE is_active = TRUE AND start_date = '...'   ✅ (첫 + 두 번째)
  WHERE is_active = TRUE AND start_date = '...' AND end_date = '...'  ✅ (전체)

이 인덱스가 동작하지 않는 쿼리:
  WHERE start_date = '...'   ❌ (첫 번째 컬럼 빠짐)
  WHERE end_date = '...'     ❌ (첫 번째 컬럼 빠짐)

→ 복합 인덱스는 왼쪽부터 순서대로 써야 효과 있음
```

---

## Partial Index — 불필요한 행은 인덱스에서 제외

```sql
-- week_rank 가 NULL 인 행은 조회할 일이 없음
-- → 인덱스 크기를 줄여 효율 높임
CREATE INDEX idx_ex_week_rank
    ON raw_exhibitions (week_rank)
    WHERE week_rank IS NOT NULL;

-- is_active = TRUE 인 것만 자주 조회
CREATE INDEX idx_ex_active
    ON raw_exhibitions (location)
    WHERE is_active = TRUE;
```

---

## 언제 만들고 언제 만들지 말아야 하나

|만들어야 할 때|만들지 말아야 할 때|
|---|---|
|WHERE 에 자주 등장하는 컬럼|데이터가 적은 테이블 (수백 건 이하)|
|JOIN 의 ON 절에 쓰이는 컬럼|거의 조회 안 하는 컬럼|
|ORDER BY 에 자주 쓰이는 컬럼|카디널리티가 낮은 컬럼 (TRUE/FALSE 같은 것)|
|카디널리티가 높은 컬럼 (값 종류가 많을수록 효과적)|모든 컬럼에 다 만들면 쓰기 속도 저하|

```
카디널리티(Cardinality) = 컬럼의 값 종류 수

높음: exhibition_id (값이 전부 다름) → 인덱스 효과 최대
낮음: is_active (TRUE/FALSE 두 가지) → 인덱스 효과 적음
      → 낮은 카디널리티는 Partial Index 로 보완
```

---

## 단점 — 공짜가 아니다

```
INSERT / UPDATE / DELETE 할 때
데이터 변경 → 인덱스도 같이 갱신 필요
→ 쓰기 속도 감소 + 저장 공간 추가 사용

실무 원칙:
  조회가 압도적으로 많은 컬럼만 인덱스 생성
  크롤러처럼 INSERT 가 잦은 경우 인덱스 최소화
```

---

## 인덱스가 실제로 쓰이는지 확인 — EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT * FROM raw_exhibitions
WHERE is_active = TRUE
  AND start_date <= CURRENT_DATE
  AND end_date >= CURRENT_DATE;
```

```
출력 예시:
  Index Scan using idx_ex_active_period on raw_exhibitions
  → ✅ 인덱스 사용 중

  Seq Scan on raw_exhibitions
  → ❌ 인덱스 무시하고 전체 스캔 중
     (데이터가 너무 적거나, 조건이 인덱스와 안 맞을 때)
```
---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|테이블이 안 생김|볼륨이 이미 있어서 init.sql 미실행|`docker compose down -v` 후 재실행|
|init.sql 수정했는데 반영 안 됨|볼륨이 이미 존재|psql 로 `ALTER TABLE` 직접 실행 + init.sql 도 수정|
|DataGrip 에 새 테이블 안 보임|캐시 문제|DataGrip 좌측 패널 F5 새로고침|
|DataGrip 연결 실패|포트포워딩 안 됨|`docker-compose.yml` 포트 확인|
|psql 접속 안 됨|컨테이너 아직 healthy 아님|`docker compose ps` 상태 확인 후 재시도|
|ENUM 타입 에러|`CREATE TYPE` 을 테이블보다 나중에 선언|init.sql 에서 `CREATE TYPE` 을 `CREATE TABLE` 위에 배치|
|Airflow DB 연결 실패|`CREATE DATABASE airflow` 누락|아래 상세 참고|
|`GRANT` 했는데 테이블 접근 불가|DB 권한과 테이블 권한 별개|`GRANT ON DATABASE` + `GRANT ON ALL TABLES` 둘 다 필요|


---
---
# Airflow DB 연결 실패 상세

```
에러:
  sqlalchemy.exc.OperationalError: (psycopg2.OperationalError)
  connection to server at "postgres" (172.18.0.4), port 5432 failed:
  FATAL: password authentication failed for user "airflow"

원인:
  Airflow 가 메타데이터 저장용으로 "airflow" DB / USER 를 사용
  PostgreSQL 에 airflow 유저와 DB 가 없으면 연결 실패
```

```
해결 순서:
  Step 1: airflow DB / USER 생성  ← 방법 A 또는 B 중 선택
  Step 2: Airflow 재시작
  Step 3: admin 계정 생성         ← 반드시 Step 1 이후
  Step 4: UI 접속
```

## Step 1-A — init.sql 에 추가 (권장)

```sql
-- postgres/init.sql 맨 위에 추가
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```

```bash
# Step 2: 볼륨 초기화 후 재실행
docker compose down -v
docker compose up -d
```

## Step 1-B — psql 에서 직접 실행 (볼륨 유지)


```bash
# 기존 데이터 유지하면서 airflow DB 만 추가
docker exec -it train-postgres psql -U train_user -d train_db

# psql 프롬프트에서 실행
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
\q
```

```
1-A vs 1-B:
  1-A  init.sql 수정 + down -v → 깔끔 / 기존 데이터 삭제
  1-B  psql 직접 실행          → 기존 데이터 유지
  → 1-B 선택 시 init.sql 도 수정해둘 것 (down -v 시 재생성 대비)
```

## Step 3 — admin 계정 생성

```
Step 1 완료 후 반드시 실행
admin 계정 없으면 Airflow UI 로그인 불가
컨테이너명은 docker-compose.yml 의 container_name
```

```bash
docker exec -it [컨테이너명] \
  airflow users create \
  --username admin \
  --password admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email admin@example.com
```

```
정상 출력:
  [INFO] Added user admin
  User "admin" created with role "Admin"

경고 메시지 (무시해도 됨):
  UserWarning: Using the in-memory storage for tracking rate limits
  → 개발 환경에서는 무시 / 운영 환경에서는 Redis 스토리지 설정 권장
```

## Step 4 — UI 접속

```
http://localhost:8084
ID: admin / PW: admin
```

```
전체 순서 정리:
  1. init.sql 에 airflow DB / USER 추가
  2. docker compose down -v && docker compose up -d
  3. airflow users create 로 admin 계정 생성
  4. http://localhost:8084 접속 → admin / admin 로그인
```