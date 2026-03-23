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

# 예시 (train 프로젝트)
docker exec -it train-postgres psql -U train_user -d train_db
#               ↑ 컨테이너 이름    ↑ DB 유저       ↑ DB 이름
#               (Docker)          (.env)            (.env)
```

```
⚠️ container_name ≠ DB명
  train-postgres  → Docker 컨테이너 이름
  train_db        → PostgreSQL 안의 데이터베이스 이름
```

```bash
# 접속 성공 시 프롬프트
train_db=#    ← 여기서 SQL 입력
```

```sql
-- 테이블 목록
\dt

-- 테이블 컬럼 구조 확인
\d 테이블명

-- 데이터 확인
SELECT * FROM 테이블명 LIMIT 10;

-- 현재 접속 정보
\conninfo

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