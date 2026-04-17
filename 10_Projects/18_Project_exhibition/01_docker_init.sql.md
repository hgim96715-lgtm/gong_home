---
aliases:
  - crawl
tags:
  - Project
related:
  - "[[00_exhibition_Project]]"
  - "[[SQL_DDL_Create]]"
  - "[[PostgreSQL_Setup]]"
  - "[[SQL_View]]"
  - "[[SQL_Data_Types]]"
---
```bash
docker exec -it exhibition-postgres psql -U exhibition_user -d exhibition_db
```

# 01_docker_init.sql — 인터파크 전시 프로젝트 인프라

## 한 줄 요약

```
Docker Compose 로 PostgreSQL / Airflow / Superset / dbt 를 띄우고
init.sql 로 테이블 스키마를 자동 생성하는 환경 설정
```

---

---

# ① 서비스 구성

|서비스|이미지|호스트 포트|역할|
|---|---|---|---|
|postgres|postgres:16|**5435**|원본 데이터 저장소|
|airflow|apache/airflow:2.9.1|**8085**|크롤링 + dbt 자동화 스케줄러|
|superset|apache/superset:3.1.0|**8090**|대시보드 시각화|
|dbt|(Dockerfile 빌드)|**8585**|데이터 변환 + 문서화|

```
포트 구분:
  5435 → PostgreSQL (로컬 5432 충돌 방지)
  8085 → Airflow    (기본 8080 충돌 방지)
  8090 → Superset   (기본 8088 충돌 방지)
  8585 → dbt docs serve
```

---

---

# ② 네트워크 & 볼륨

```yaml
networks:
  exhibition-network:
    driver: bridge
# 모든 서비스가 같은 네트워크 → 서비스명으로 통신 가능
# postgres 컨테이너에 접근할 때 host = "postgres" (IP 아님)

volumes:
  postgres_data:   # DB 데이터 영구 보존
  superset_data:   # Superset 차트/대시보드 보존
```

```
⚠️ docker compose down -v 하면
  postgres_data + superset_data 전부 삭제
  → 차트, 대시보드, DB 데이터 모두 날아감
  → 개발 초기 스키마 세팅 단계에서만 사용
```

---

---

# ③ PostgreSQL 서비스

```yaml
postgres:
  image: postgres:16
  container_name: exhibition-postgres
  ports:
    - "5435:5432"           # 호스트:컨테이너
  environment:
    - POSTGRES_USER=${POSTGRES_USER}
    - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    - POSTGRES_DB=${POSTGRES_DB}
  volumes:
    - postgres_data:/var/lib/postgresql/data
    - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
    interval: 10s
    retries: 5
```

```
init.sql 자동 실행 조건:
  docker-entrypoint-initdb.d/ 안의 .sql 은
  볼륨이 비어있을 때 (최초 실행) 딱 한 번 자동 실행

⚠️ 볼륨이 이미 있으면 init.sql 재실행 안 됨
  스키마 수정 시:
    방법 A: docker compose down -v → 재실행 (데이터 삭제)
    방법 B: psql 에서 ALTER TABLE 직접 실행 (데이터 유지)
           → init.sql 도 동일하게 수정해둘 것
```

---

---

# ④ Airflow 서비스

```yaml
airflow:
  image: apache/airflow:2.9.1
  depends_on:
    postgres:
      condition: service_healthy   # postgres 준비 완료 후 시작
  environment:
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: >
      postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/airflow
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    # DAG 에서 쓸 DB 연결 정보
    POSTGRES_HOST: postgres        # 서비스명 (컨테이너 내부 통신)
    POSTGRES_PORT: 5432            # 컨테이너 내부 포트
  volumes:
    - ./airflow/dags:/opt/airflow/dags
    - ./crawl:/opt/airflow/crawl
    - ./load:/opt/airflow/load
    - ./dbt_exhibition:/opt/airflow/dbt
```

```
Airflow 메타데이터 DB:
  airflow DB 가 PostgreSQL 에 있어야 함
  init.sql 맨 위에 CREATE DATABASE airflow; 선언

airflow 컨테이너 안에서 PostgreSQL 접근:
  host = "postgres"  (서비스명)
  port = 5432        (컨테이너 내부 포트)
  ← 로컬 머신에서 접근할 때의 5435 가 아님!
```

---

---

# ⑤ dbt 서비스

```yaml
dbt:
  build: .                         # 루트의 Dockerfile 로 빌드
  ports:
    - "8585:8585"
  working_dir: /dbt_exhibition
  volumes:
    - ./dbt_exhibition:/dbt_exhibition
  depends_on:
    postgres:
      condition: service_healthy
  command: tail -f /dev/null       # 컨테이너 유지 (대기 상태)
```

```bash
# dbt 실행 방법
docker exec -it exhibition-dbt bash

dbt debug          # 연결 확인
dbt run            # 전체 모델 실행
dbt test           # 테스트
dbt docs generate  # 문서 생성
dbt docs serve --host 0.0.0.0 --port 8585  # http://localhost:8585
```

---

---

# ⑥ init.sql — 테이블 스키마

## 테이블 4개

```
raw_exhibitions         ← 전시 원본 (Summary API + Place API)
raw_exhibition_prices   ← 가격 상세 (/prices/group 정규화)
raw_exhibition_stats    ← 관람객 통계 (/statistics/booking)
raw_exhibition_history  ← 순위·가격 일별 스냅샷
```

## raw_exhibitions 핵심 컬럼

```sql
exhibition_id  VARCHAR(50)  UNIQUE NOT NULL  -- goodsCode (PK)
title          VARCHAR(500) NOT NULL          -- 원본 제목 (접두어 포함)
location       VARCHAR(50)                   -- 시/도 단위 (LOCATION_MAP 정규화)
address        VARCHAR(500)                  -- Place API placeAddress
latitude       NUMERIC(10, 7)               -- Place API
longitude      NUMERIC(10, 7)               -- Place API
day_rank       INTEGER                       -- dayRank (숫자)
week_rank      INTEGER                       -- weekRank (숫자)
month_rank     INTEGER                       -- monthRank (숫자)
rank           VARCHAR(100)                  -- "주간 1위 / 일간 1위" 텍스트
prices_raw     JSONB                         -- 가격 API 원문 → dbt 에서 파싱
is_active      BOOLEAN      DEFAULT TRUE     -- 크롤링 수집 여부
```

## raw_exhibition_prices — UNIQUE 제약 필수

```sql
UNIQUE (exhibition_id, seat_grade, price_grade, price_type_code)
-- ON CONFLICT (exhibition_id, seat_grade, price_grade, price_type_code) 에 필요
-- 없으면 upsert_exhibition_prices 실행 시 에러
```

## 인덱스 전략

|인덱스|컬럼|이유|
|---|---|---|
|`idx_ex_location`|location|지역별 필터|
|`idx_ex_category`|category|카테고리 필터|
|`idx_ex_active_period`|is_active, start_date, end_date|진행중 조회 복합|
|`idx_ex_week_rank`|week_rank (WHERE NOT NULL)|순위 조회 Partial Index|
|`idx_prices_exhibition`|exhibition_id|FK 조인|
|`idx_history_date`|snapshot_date|날짜별 스냅샷 조회|

---

---

# ⑦ 환경변수 (.env)

```
⚠️ 포트 주의:
  크롤러 (로컬 실행)  → POSTGRES_PORT=5435 (호스트 포트)
  Airflow DAG (컨테이너 안) → POSTGRES_PORT=5432 (내부 포트)
  → 환경변수를 다르게 설정하거나 DAG 에서 직접 5432 지정
```

---

---

# ⑧ 자주 쓰는 명령어

```bash
# 전체 실행
docker compose up -d

# 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f postgres
docker compose logs -f airflow

# PostgreSQL 접속
docker exec -it exhibition-postgres psql -U exhibition_user -d exhibition_db

# 스키마 변경 (볼륨 유지)
docker exec -it exhibition-postgres psql -U exhibition_user -d exhibition_db \
  -c "ALTER TABLE raw_exhibitions ADD COLUMN IF NOT EXISTS new_col TEXT;"

# 전체 초기화 (데이터 삭제)
docker compose down -v
docker compose up -d
```

---

---

