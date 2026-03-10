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
---

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

```text
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

> ⚠️ `down -v` 는 기존 데이터도 전부 날아간다. 운영 중인 DB 에서는 절대 사용 금지. 
> 개발 초기 스키마 세팅 단계에서만 사용할 것.

---

---

# ② 자주 쓰는 자료형

| 자료형            | 설명              | 예시                    |
| -------------- | --------------- | --------------------- |
| `SERIAL`       | 자동 증가 정수 (PK 용) | 1, 2, 3 ...           |
| `VARCHAR(n)`   | 최대 n 자리 문자열     | `VARCHAR(20)`         |
| `TEXT`         | 길이 제한 없는 문자열    | 긴 설명 등                |
| `INTEGER`      | 정수              | 100, 200              |
| `NUMERIC(p,s)` | 소수점 포함 숫자       | 금액, 좌표                |
| `TIMESTAMP`    | 날짜 + 시각         | `2026-03-05 21:00:00` |
| `DATE`         | 날짜만             | `2026-03-05`          |
| `BOOLEAN`      | 참/거짓            | `TRUE`, `FALSE`       |
| `ENUM`         | 미리 정의한 고정 값 목록  | `'정시'`, `'지연'`        |

```sql
-- 이 프로젝트에서 쓰는 자료형 예시
run_ymd           VARCHAR(8)    -- '20260305' 형태 문자열로 받음
trn_plan_dptre_dt VARCHAR(20)   -- '20260305210000' 형태 문자열로 받음
created_at        TIMESTAMP DEFAULT NOW()  -- 저장 시각 자동 기록
dep_status        delay_status  -- ENUM 타입 (정시 / 소폭지연 / 지연 / 대폭지연)
```

```sql
-- ENUM 타입은 CREATE TABLE 전에 먼저 선언해야 함
CREATE TYPE delay_status AS ENUM ('정시', '소폭지연', '지연', '대폭지연');

-- 이후 컬럼에 바로 사용 가능
dep_status  delay_status,
arr_status  delay_status
```

>ENUM 상세 → [[SQL_DDL_Create#② - 2. ENUM 타입 — 고정된 값 목록|ENUM 타입]]

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

```sql
-- 테이블 목록
\dt

-- 테이블 컬럼 구조 확인
\d 테이블명

-- 데이터 확인
SELECT * FROM 테이블명 LIMIT 10;

-- 접속 종료
\q
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|테이블이 안 생김|볼륨이 이미 있어서 init.sql 미실행|`docker compose down -v` 후 재실행|
|init.sql 수정했는데 반영 안 됨|볼륨이 이미 존재해서 재실행 안 함|`down -v` → init.sql 수정 → `up -d` → DataGrip 새로고침|
|DataGrip 에 새 테이블이 안 보임|캐시 문제|DataGrip 좌측 패널에서 F5 또는 새로고침 클릭|
|DataGrip 연결 실패|포트포워딩 안 됨|`docker-compose.yml` 에서 포트 확인|
|`psql` 접속 안 됨|컨테이너가 아직 healthy 아님|`docker compose ps` 로 상태 확인 후 재시도|
|ENUM 타입 에러|`CREATE TYPE` 을 테이블보다 나중에 선언|init.sql 에서 `CREATE TYPE` 을 `CREATE TABLE` 위에 배치|


---
