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

## 개념 한 줄 요약

> **"PostgreSQL 을 Docker 로 띄우고, DataGrip 으로 연결해서 바로 쓰는 환경 세팅."**

---

---

# ① init.sql — 컨테이너 최초 실행 시 테이블 자동 생성

> `docker-entrypoint-initdb.d/` 안에 마운트된 `.sql` 파일은 컨테이너 **최초 실행 시 딱 한 번** 자동으로 실행된다.

```yaml
# docker-compose.yml 에서 마운트
volumes:
  - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```

```
동작 순서:
docker compose up -d
  → PostgreSQL 컨테이너 시작
  → 볼륨이 비어있으면 init.sql 자동 실행
  → 테이블 생성 완료
```

```
⚠️  init.sql 이 실행 안 될 때:
이미 볼륨이 존재하면 init.sql 을 다시 실행하지 않는다.

해결:
$ docker compose down -v   ← 볼륨까지 삭제
$ docker compose up -d     ← 다시 올리면 init.sql 실행됨
```

```
💡 init.sql 에 테이블을 새로 추가했다면?
  → 파일만 수정해도 바로 반영 안 됨 (볼륨이 이미 있으니까)
  → docker compose down -v 로 초기화 후 다시 up 하면 나옴

순서:
$ docker compose down -v   ← 기존 볼륨(데이터) 전부 삭제
  (init.sql 수정)
$ docker compose up -d     ← 컨테이너 재시작 → init.sql 다시 실행
  DataGrip 에서 새로고침 (F5 또는 좌측 새로고침 버튼)
  → 새 테이블 확인
```

> ⚠️ `down -v` 는 기존 데이터도 전부 날아간다. 운영 중인 DB 에서는 절대 사용 금지. 개발 초기 스키마 세팅 단계에서만 사용할 것.
> `down -v`없이 수정은 [[PostgreSQL_Setup#⑥ 스키마 수정 — down -v 없이 변경하기 ⭐️|⑥ 스키마 수정 — down -v 없이 변경하기 ⭐️]] 확인 

---

---

# ② 자주 쓰는 자료형

|자료형|설명|예시|
|---|---|---|
|`SERIAL`|자동 증가 정수 (PK 용)|1, 2, 3 ...|
|`VARCHAR(n)`|최대 n 자리 문자열|`VARCHAR(20)`|
|`TEXT`|길이 제한 없는 문자열|긴 설명 등|
|`INTEGER`|정수|100, 200|
|`NUMERIC(p,s)`|소수점 포함 숫자|금액, 좌표|
|`TIMESTAMP`|날짜 + 시각|`2026-03-05 21:00:00`|
|`DATE`|날짜만|`2026-03-05`|
|`BOOLEAN`|참/거짓|`TRUE`, `FALSE`|
|`ENUM`|미리 정의한 고정 값 목록|`'정시'`, `'지연'`|

```sql
-- 이 프로젝트에서 쓰는 자료형 예시
run_ymd           VARCHAR(8)    -- '20260305' 형태 문자열로 받음
trn_plan_dptre_dt VARCHAR(20)   -- '20260305210000' 형태 문자열로 받음
created_at        TIMESTAMP DEFAULT NOW()  -- 저장 시각 자동 기록
dep_status        delay_status  -- ENUM 타입 (정시 / 소폭지연 / 지연 / 대폭지연)
```

```sql
-- delay_label() 이 '정시(0분)', '소폭지연(+3분)' 처럼 숫자 포함 텍스트를 반환시에는  ENUM 미사용
-- → ENUM 고정값과 불일치 → dep_status / arr_status 를 VARCHAR(50) 으로 변경

dep_status   VARCHAR(50),   -- 예: '정시(0분)', '대폭지연(+45분)'
arr_status   VARCHAR(50),
```

> ENUM 개념 상세 → [[SQL_DDL_Create#② - 2. ENUM 타입 — 고정된 값 목록|ENUM 타입]]

---

---

# ③ DataGrip 연결

> `docker compose up -d` 후 DataGrip 에서 바로 연결 가능.

```
DataGrip → New → Data Source → PostgreSQL

Host:      localhost
Port:      5433
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

> 처음 연결 시 드라이버 다운로드 팝업이 뜨면 **Download** 클릭.

## ⚠️ DataGrip 에서 "user does not exist" 에러가 나면?

```
원인: 로컬 5432 포트가 이미 사용 중 → 컨테이너가 제대로 안 뜬 것
```

```bash
# 1. 로컬에서 5432 포트 사용 중인지 확인
$ lsof -i :5432
```

```
결과에 뭔가 뜨면 → 로컬 PostgreSQL 이 5432 를 이미 점유 중
→ docker-compose.yml 에서 호스트 포트를 5433 으로 변경
```

```yaml
# docker-compose.yml
ports:
  - "5433:5432"  # 로컬 5432 충돌로 인해 호스트 포트를 5433 으로 변경
```

```
변경 후 DataGrip 연결 정보도 Port: 5433 으로 수정
컨테이너 내부는 여전히 5432 → 컨테이너끼리 통신할 때는 5432 그대로 사용
```

---

---

# ④ psql 기본 명령어

> 터미널에서 컨테이너 직접 접속해서 확인하는 방법.

```bash
# 컨테이너 내부 psql 접속
$ docker exec -it <container_name> psql -U <유저명> -d <DB명>
```

```text
옵션 분해:
  docker exec   실행 중인 컨테이너 안에서 명령어 실행
  -it           터미널 입출력 연결 (interactive + tty)
  container_name  docker-compose.yml 의 container_name 값
  psql          PostgreSQL 클라이언트 실행
  -U            유저명 (.env 의 POSTGRES_USER)
  -d            접속할 DB명 (.env 의 POSTGRES_DB)

헷갈리는 부분:
  container_name ≠ DB명
  train-postgres = 컨테이너 이름 (도커 컨테이너)
  train_db       = DB 이름 (PostgreSQL 안의 데이터베이스)
```

```bash
# .env 기준
# POSTGRES_USER=train_user
# POSTGRES_PASSWORD=train_password
# POSTGRES_DB=train_db
# container_name: train-postgres

docker exec -it train-postgres psql -U train_user -d train_db
#               ↑ 컨테이너 이름       ↑ DB 유저     ↑ DB 이름
#               (도커)               (.env)         (.env)
```

```bash
# 접속 확인 — 아래처럼 나오면 성공 # psql (16.x) # Type "help" for help. # train_db=# ← 프롬프트에 DB명이 보임
```

```sql
-- 테이블 목록
\dt

-- 테이블 컬럼 구조 확인
-- 얘시) \d train_realtime
\d 테이블명

-- 데이터 확인
SELECT * FROM 테이블명 LIMIT 10;

-- 현재 접속 정보
\conninfo

-- 접속 종료
\q
```

---

---

# ⑤ DB · USER · 권한 관리

```
새 프로젝트 시작 흐름:
  CREATE DATABASE  → DB 만들기
  CREATE USER      → 전용 유저 만들기
  GRANT            → 권한 주기
  ALTER DATABASE   → 소유권 설정
  \l / \du / \dp   → 확인
```

## DB 생성 / 삭제

```sql
-- DB 생성
CREATE DATABASE mydb;

-- DB 삭제 (접속 중인 유저 없어야 함)
DROP DATABASE mydb;

-- DB 목록 확인 (psql 메타 명령어)
\l
```

## USER 생성 / 삭제

```sql
-- 유저 생성
CREATE USER myuser WITH PASSWORD 'mypassword';

-- 유저 생성 + 로그인 권한 명시
CREATE USER myuser WITH LOGIN PASSWORD 'mypassword';

-- 유저 삭제
DROP USER myuser;

-- 비밀번호 변경
ALTER USER myuser WITH PASSWORD 'newpassword';

-- 유저 목록 확인 (psql 메타 명령어)
\du
```

## GRANT — 권한 부여

```sql
-- DB 전체 접근 권한
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

-- 특정 테이블만
GRANT SELECT, INSERT ON TABLE mytable TO myuser;

-- 모든 테이블
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO myuser;

-- 소유권 이전 (DB 소유자 변경)
ALTER DATABASE mydb OWNER TO myuser;

-- 권한 회수
REVOKE ALL PRIVILEGES ON DATABASE mydb FROM myuser;
```

## 실전 패턴 — 프로젝트 전용 DB + 유저 세팅

```sql
-- init.sql 에 추가하는 패턴 (프로젝트마다 독립된 DB/유저)
CREATE DATABASE airflow;
CREATE USER airflow WITH PASSWORD 'airflow';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
ALTER DATABASE airflow OWNER TO airflow;
```

```
각 명령어 역할:
  CREATE DATABASE airflow           airflow 용 DB 생성
  CREATE USER airflow               전용 유저 생성
  GRANT ALL PRIVILEGES ON DATABASE  DB 접근 + 테이블 생성 등 전체 권한
  ALTER DATABASE OWNER              소유권까지 넘김 → 완전한 독립 환경
```

## 권한 확인 명령어

```sql
-- DB 목록 + 소유자 + 권한
\l

-- 유저 목록 + 역할(ROLE) + 슈퍼유저 여부
\du

-- 테이블 권한 목록
\dp

-- 현재 접속 정보
\conninfo

-- 특정 DB 로 전환
\c mydb
```

## 스키마(Schema) — public 이 뭔가?

```
스키마 = DB 안에 있는 폴더 (테이블을 묶는 네임스페이스)

PostgreSQL 기본 스키마: public
  → 별도 지정 없으면 모든 테이블이 public 스키마에 생성

Superset Dataset 등록 시 "스키마 = public" 선택하는 이유가 이것
```

```sql
-- 스키마 목록
\dn

-- 현재 search_path 확인 (기본 스키마 탐색 순서)
SHOW search_path;

-- 스키마 명시해서 조회
SELECT * FROM public.train_realtime;

-- search_path 변경 (세션 동안)
SET search_path TO myschema, public;
```

---

---

# ⑥ 스키마 수정 — down -v 없이 변경하기 ⭐️

```
down -v 는 볼륨 전체 삭제
  → PostgreSQL 데이터 삭제
  → Superset 차트·대시보드·DB연결 전부 삭제
  → 절대 일상적으로 쓰면 안 됨

실제로 테이블 구조를 수정해야 할 때는
psql 로 직접 들어가서 ALTER TABLE 로 처리
```

## 컨테이너 안으로 직접 접속

```bash
# postgres 컨테이너 내부 psql 접속
#$ docker exec -it <container_name> psql -U <유저명> -d <DB명>
docker exec -it train-postgres psql -U postgres -d train_db

# 접속 확인
\conninfo
# You are connected to database "train_db" as user "postgres"
```

## 테이블 구조 변경 — ALTER TABLE

```sql
-- 컬럼 추가
ALTER TABLE train_realtime ADD COLUMN new_col VARCHAR(50);

-- 컬럼 삭제
ALTER TABLE train_realtime DROP COLUMN old_col;

-- 컬럼 타입 변경
ALTER TABLE train_realtime ALTER COLUMN status TYPE TEXT;

-- 컬럼 이름 변경
ALTER TABLE train_realtime RENAME COLUMN old_name TO new_name;

-- NOT NULL 제약 추가
ALTER TABLE train_realtime ALTER COLUMN trn_no SET NOT NULL;

-- NOT NULL 제약 제거
ALTER TABLE train_realtime ALTER COLUMN trn_no DROP NOT NULL;

-- 기본값 설정
ALTER TABLE train_realtime ALTER COLUMN data_type SET DEFAULT 'estimated';
```

## 데이터 수정 — UPDATE / DELETE

```sql
-- 특정 조건 데이터 수정
UPDATE train_realtime
SET status = '운행 중'
WHERE status IS NULL;

-- 테이블 데이터 전체 삭제 (구조는 유지)
TRUNCATE TABLE train_realtime;

-- 조건부 삭제
DELETE FROM train_realtime WHERE created_at < NOW() - INTERVAL '7 days';
```

## init.sql 도 같이 수정해두기 ⭐️

```
psql 로 직접 수정하면 현재 실행 중인 DB 는 반영됨
BUT init.sql 은 그대로 → 나중에 down -v 하면 수정 전 상태로 돌아감

→ psql 로 수정 후 반드시 init.sql 도 동일하게 수정해둘 것
```

```
수정 작업 순서:

1. psql 접속 → ALTER TABLE 실행 (즉시 반영)
2. init.sql 도 동일하게 수정 (미래 재구성 대비)
3. DataGrip F5 새로고침
4. Superset Dataset → Edit → Sync columns from source
```

## down -v 가 필요한 경우 vs 불필요한 경우

```
down -v 필요 (볼륨 재생성이 필수):
  init.sql 에 CREATE TYPE / ENUM 추가 (타입은 ALTER 로 추가 불가)
  DB 자체를 새로 만들어야 할 때
  볼륨이 꼬여서 컨테이너가 아예 안 올라올 때

down -v 불필요 (psql 직접 수정):
  컬럼 추가 / 삭제 / 타입 변경    → ALTER TABLE
  데이터 수정 / 삭제              → UPDATE / DELETE / TRUNCATE
  인덱스 추가                     → CREATE INDEX
  새 테이블 추가                  → CREATE TABLE (psql 에서 바로 실행)
```

|실수|원인|해결|
|---|---|---|
|테이블이 안 생김|볼륨이 이미 있어서 init.sql 미실행|`docker compose down -v` 후 재실행|
|init.sql 수정했는데 반영 안 됨|볼륨이 이미 존재해서 재실행 안 함|psql 로 직접 `ALTER TABLE` 실행 + init.sql 도 같이 수정 (⑥ 참고)|
|DataGrip 에 새 테이블이 안 보임|캐시 문제|DataGrip 좌측 패널에서 F5 또는 새로고침 클릭|
|DataGrip 연결 실패|포트포워딩 안 됨|`docker-compose.yml` 에서 포트 확인|
|`psql` 접속 안 됨|컨테이너가 아직 healthy 아님|`docker compose ps` 로 상태 확인 후 재시도|
|ENUM 타입 에러|`CREATE TYPE` 을 테이블보다 나중에 선언|init.sql 에서 `CREATE TYPE` 을 `CREATE TABLE` 위에 배치|
|Airflow DB 연결 실패|`CREATE DATABASE airflow` 누락|init.sql 에 DB 생성 + 유저 + GRANT 3줄 추가|
|`GRANT` 했는데 테이블 접근 불가|DB 권한과 테이블 권한이 별개|`GRANT ON DATABASE` + `GRANT ON ALL TABLES` 둘 다 필요|
|`\l` 에서 스키마 선택 안 됨|Superset Dataset 등록 시 스키마 미선택|`public` 스키마 명시적으로 선택|

---

## 관련 노트

- [[00_SQL_HomePage]] — SQL 홈
- [[00_Docker_HomePage]] — Docker 홈
- [[Docker_Volumes]] — 볼륨 down -v 원리
- [[SQL_DDL_Create]] — CREATE TABLE / ENUM / CHECK 문법
- [[SQL_DCL_Grant_Revoke]] — 테이블 수준 GRANT/REVOKE
- [[06_Airflow_Pipeline]] — init.sql 에 airflow DB 추가하는 이유
- [[16_Project(Train)/01_Docker_Setup]] — 실전 프로젝트 적용