---
aliases:
  - Hospital Airflow
  - er_hospitals DAG
  - er_hourly_stats DAG
tags:
  - Project
related:
  - "[[00_Hospital_Project]]"
  - "[[PostgreSQL_Setup]]"
  - "[[Python_Database_Connect]]"
  - "[[Serialization_JSON_XML]]"
  - "[[Python_ElementTree]]"
  - "[[Python_Requests_Methods]]"
  - "[[Airflow_Hooks]]"
  - "[[Airflow_DAG_Skeleton]]"
  - "[[SQL_DML_CRUD]]"
---
# 06_Hospital_Airflow — 배치 파이프라인

## 이 단계의 목표

```
① Airflow 컨테이너 세팅 + 접속 확인
② er_hospitals 수집 DAG  — 병원 기본정보 1일 1회
③ er_hourly_stats 집계 DAG — 시간대별 병상 통계
④ Superset 으로 돌아가서 지역별 차트 / 지도 추가
```

```
왜 Airflow 가 필요한가:
  er_hospitals (병원 기본정보) → 자주 안 바뀜 → 1일 1회 배치로 충분
  Kafka/Spark 로 실시간 수집 → 오버엔지니어링

  er_hourly_stats (시간대별 집계) → Superset 쿼리 최적화 용도
  er_realtime 직접 쿼리 → 데이터 많아지면 느려짐
  → Airflow 로 미리 집계해두면 Superset 빠름

  "실시간(Kafka/Spark) + 배치(Airflow)" 조합
  → 현업에서 실제로 쓰는 Lambda Architecture
```

---

---

# ① 사전 준비 — 실행 순서

```
Airflow DAG 를 실행하기 전에
아래 서비스들이 먼저 실행 중이어야 함
```

```bash
# 1. Docker 서비스 전체 실행
docker compose up -d

# 2. 실행 상태 확인
docker compose ps

# 3. Producer 실행 (er_realtime 데이터가 있어야 집계 가능)
cd producer && python3 produer.py

# 4. Consumer 실행 (Kafka → er_realtime 적재)
docker exec -it hospital-spark-master \ /opt/spark/bin/spark-submit \ --master spark://spark-master:7077 \ --executor-memory 1g \ --driver-memory 1g \ --jars /opt/spark/jars/spark-sql-kafka-0-10_2.12-3.5.0.jar,/opt/spark/jars/spark-token-provider-kafka-0-10_2.12-3.5.0.jar,/opt/spark/jars/kafka-clients-3.4.1.jar,/opt/spark/jars/commons-pool2-2.11.1.jar,/opt/spark/jars/postgresql-42.7.1.jar \ /opt/spark/apps/consumer.py
```

```
실행 순서 요약:
  1. docker compose up -d
  2. Producer 실행  → Kafka 에 데이터 발행
  3. Consumer 실행  → Kafka → er_realtime 적재
  4. Airflow DAG 실행 → er_hospitals 수집 + er_hourly_stats 집계
  5. Superset 에서 차트 확인

  Producer / Consumer 없이 DAG 실행 시:
    er_realtime 데이터 없음 → er_hourly_stats 집계 결과 0건
```

---

---

# ② Airflow 접속

> 사전 준비 → [[PostgreSQL_Setup#Airflow DB 연결 실패 상세]] 참고

```bash
# 브라우저 접속
http://localhost:8084

# 초기 계정
ID:       admin
Password: admin
```

---

---

# ③ er_hospitals 수집 DAG

## 개요

```
path:   getEgytBassInfoInqire
주기:   1일 1회 (새벽 5시)
테이블: er_hospitals
방식:   ON CONFLICT (hpid) DO UPDATE (UPSERT)
        → 매일 실행해도 중복 없음 / 변경사항 자동 반영
```

## DAG 코드 — hospital_info_dag.py

```python
import os
import requests
import xml.etree.ElementTree as ET
from datetime import datetime
from urllib.parse import unquote
import pendulum

from airflow.decorators import dag, task
from airflow.providers.postgres.hooks.postgres import PostgresHook
from psycopg2.extras import execute_values

API_KEY  = unquote(os.getenv("HOSPITAL_API_KEY", ""))
BASE_URL = "http://apis.data.go.kr/B552657/ErmctInfoInqireService"
SUCCESS_CODES = {"00", "0000"}

def parse_float(val, default=None):
    try:   return float(val) if val else default
    except: return default

def parse_int(val, default=None):
    try:   return int(val) if val else default
    except: return default

def get_region(addr: str) -> str:
    return addr[:2] if addr else ""

def strip_str(val, default=""):
    return str(val).strip() if val else default


default_args = {
    "owner":             "hospital",
    "retries":           1,
    "retry_delay":       pendulum.duration(minutes=5),
    "execution_timeout": pendulum.duration(minutes=10),
    "depends_on_past":   False,
    "email_on_failure":  True,
}

@dag(
    dag_id          = "hospital_info_dag",
    default_args    = default_args,
    description     = "응급의료기관 기본정보 1일 1회 수집",
    schedule_interval = "0 5 * * *",
    start_date      = pendulum.datetime(2026, 3, 23, tz="Asia/Seoul"),
    catchup         = False,
    tags            = ["hospital", "batch"],
)
def hospital_info_pipeline():

    @task
    def fetch_hospitals():
        params = {"serviceKey": API_KEY, "numOfRows": 1000, "pageNo": 1}

        try:
            res = requests.get(
                f"{BASE_URL}/getEgytBassInfoInqire",
                params=params, timeout=30
            )
            res.raise_for_status()
            root = ET.fromstring(res.content)
            result_code = root.findtext(".//resultCode", "")
            if result_code not in SUCCESS_CODES:
                raise Exception(f"API 에러: {result_code}")
        except Exception as e:
            print(f"[에러] {e}")
            raise

        rows = []
        for item in root.findall(".//item"):
            duty_addr = item.findtext("dutyAddr", "").strip()
            rows.append((
                strip_str(item.findtext("hpid")),
                strip_str(item.findtext("dutyName")),
                duty_addr,
                strip_str(item.findtext("dutyTel3")),
                strip_str(item.findtext("dutyEryn")),
                parse_float(item.findtext("wgs84Lat")),
                parse_float(item.findtext("wgs84Lon")),
                parse_int(item.findtext("hpbdn")),
                strip_str(item.findtext("MKioskTy1")),
                strip_str(item.findtext("MKioskTy3")),
                strip_str(item.findtext("MKioskTy25")),
                strip_str(item.findtext("MKioskTy10")),
                get_region(duty_addr),
            ))
        print(f"[수집] {len(rows)}개 병원")

        # ── PostgreSQL UPSERT ─────────────────────────────
        hook = PostgresHook(postgres_conn_id="postgres_hospital")
        conn = hook.get_conn()
        cur  = conn.cursor()

        sql = """
            INSERT INTO er_hospitals (
                hpid, hpname, duty_addr, duty_tel, duty_eryn,
                wgs84_lat, wgs84_lon, hpbdn,
                mk_stroke, mk_cardiac, mk_trauma, mk_pediatric,
                region, updated_at
            ) VALUES %s
            ON CONFLICT (hpid) DO UPDATE SET
                hpname       = EXCLUDED.hpname,
                duty_addr    = EXCLUDED.duty_addr,
                duty_tel     = EXCLUDED.duty_tel,
                duty_eryn    = EXCLUDED.duty_eryn,
                wgs84_lat    = EXCLUDED.wgs84_lat,
                wgs84_lon    = EXCLUDED.wgs84_lon,
                hpbdn        = EXCLUDED.hpbdn,
                mk_stroke    = EXCLUDED.mk_stroke,
                mk_cardiac   = EXCLUDED.mk_cardiac,
                mk_trauma    = EXCLUDED.mk_trauma,
                mk_pediatric = EXCLUDED.mk_pediatric,
                region       = EXCLUDED.region,
                updated_at   = NOW()
        """
        rows_with_ts = [r + (datetime.now(),) for r in rows]
        execute_values(cur, sql, rows_with_ts)
        conn.commit()
        cur.close()
        conn.close()
        print(f"[적재] {len(rows)}개 → er_hospitals UPSERT 완료")

    fetch_hospitals()

hospital_info_pipeline()
```

>[[Airflow_Hooks]] & [[Python_Database_Connect]]참고 

---

---

# ④ er_hourly_stats 집계 DAG

## 왜 필요한가

```
Superset 에서 시간대별 차트를 그리려면 er_realtime 직접 쿼리
→ 데이터 쌓일수록 느려짐

→ Airflow 로 매시간 미리 집계 → er_hourly_stats 저장
→ Superset 은 작은 집계 테이블만 조회 → 빠름

ON CONFLICT DO NOTHING 이유:
  "오늘 오후 3시의 평균 병상 수" 는 과거 사실 → 불변
  이미 집계된 데이터는 덮어쓸 필요 없음
```

## DAG 코드 — er_hourly_stats_dag.py

```python
import pendulum
from airflow.decorators import dag, task
from airflow.providers.postgres.hooks.postgres import PostgresHook

default_args = {
    "owner":       "hospital",
    "retries":     1,
    "retry_delay": pendulum.duration(minutes=3),
}

@dag(
    dag_id          = "er_hourly_stats_dag",
    default_args    = default_args,
    description     = "er_realtime 시간대별 집계",
    schedule_interval = "@hourly",
    start_date      = pendulum.datetime(2026, 3, 23, tz="Asia/Seoul"),
    catchup         = False,
    tags            = ["hospital", "batch", "mart"],
)
def hourly_stats_pipeline():

    @task
    def aggregate_hourly(data_interval_start, data_interval_end):
        hook = PostgresHook(postgres_conn_id="postgres_hospital")

        # .start_of('hour') → 수동 실행 시에도 무조건 정각으로 맞춤
        start_time = (
            data_interval_start.in_timezone("Asia/Seoul")
            .start_of("hour").strftime("%Y-%m-%d %H:%M:%S")
        )
        end_time = (
            data_interval_end.in_timezone("Asia/Seoul")
            .start_of("hour").strftime("%Y-%m-%d %H:%M:%S")
        )
        print(f"[집계 구간] {start_time} ~ {end_time}")

        sql = """
            INSERT INTO er_hourly_stats (
                stat_hour, region,
                avg_beds, zero_count, total_hospitals, saturation_pct,
                created_at
            )
            SELECT
                DATE_TRUNC('hour', r.created_at)        AS stat_hour,
                h.region,
                ROUND(AVG(r.hvec)::NUMERIC, 2)          AS avg_beds,
                COUNT(*) FILTER(WHERE r.hvec <= 0)      AS zero_count,
                COUNT(*)                                 AS total_hospitals,
                ROUND(
                    COUNT(*) FILTER(WHERE r.hvec <= 0)::NUMERIC
                    / NULLIF(COUNT(*), 0) * 100
                , 1)                                    AS saturation_pct,
                NOW() AT TIME ZONE 'Asia/Seoul'         AS created_at
            FROM er_realtime r
            JOIN er_hospitals h ON r.hpid = h.hpid
            WHERE r.created_at >= %s::timestamp
              AND r.created_at <  %s::timestamp
            GROUP BY DATE_TRUNC('hour', r.created_at), h.region
            ON CONFLICT DO NOTHING
        """
        hook.run(sql, parameters=(start_time, end_time))
        print("[집계 완료] er_hourly_stats 업데이트 완료")

    aggregate_hourly()

hourly_stats_pipeline()
```

>[[Airflow_Hooks]] & [[Python_Database_Connect]]참고 

---

---

# ⑤ 폴더 구조

```
hospital-project/
├── dags/
│   ├── hospital_info_dag.py       ← 병원 기본정보 수집
│   └── er_hourly_stats_dag.py     ← 시간대별 집계
├── docker-compose.yml
└── ...
```

```yaml
# docker-compose.yml Airflow 볼륨 마운트
services:
  airflow:
    volumes:
      - ./dags:/opt/airflow/dags
```

---

---

# ⑥ 실행 & 확인

```bash
# DAG 목록 확인
docker exec -it hospital-airflow airflow dags list

# 수동 실행 (최초 테스트)
docker exec -it hospital-airflow \
    airflow dags trigger hospital_info_dag

# 로그 확인
docker exec -it hospital-airflow \
    airflow tasks logs hospital_info_dag fetch_hospitals

# Airflow UI → http://localhost:8084 → DAG → Trigger ▶
```


---

---

# 설계 고민 노트

```
시간대 통일 (UTC → KST):
  Spark consumer.py created_at:
    from_utc_timestamp(current_timestamp(), "Asia/Seoul") 로 변경
  er_hourly_stats created_at:
    NOW() AT TIME ZONE 'Asia/Seoul' 로 저장
  → 모든 테이블 시각 KST 기준으로 통일 완료

ON CONFLICT 전략:
  er_hospitals    → DO UPDATE (병원 정보는 바뀔 수 있음)
  er_hourly_stats → DO NOTHING (과거 집계는 불변)

region 이 다 서울로 나오는 문제:
  API 응답의 dutyAddr 앞 2글자를 region 으로 사용
  현재 수집된 병원 대부분이 서울 기반 데이터인 것으로 추정
  → 전국 데이터 수집 시 자동 해소 예정
  → Superset 에서 region 필터로 확인 필요

hpbdn / mk 컬럼 NULL 많음:
  API 응답에서 hpbdn, mk_stroke, mk_cardiac, mk_trauma, mk_pediatric
  값이 없는 병원이 많음
  → NULL 그대로 저장 / Superset 에서 IS NOT NULL 필터로 처리
  → 향후 COALESCE 으로 기본값 처리 고려

mk_trauma / mk_cardiac VARCHAR(1) → VARCHAR(10) 변경:
  실제 데이터에 'N1' 같은 2자 이상 값이 들어옴
  → DDL ALTER TABLE 로 컬럼 타입 확장
  ALTER TABLE er_hospitals
      ALTER COLUMN mk_trauma  TYPE VARCHAR(10),
      ALTER COLUMN mk_cardiac TYPE VARCHAR(10);

```

---

---

# 트러블슈팅

| 증상                            | 원인                          | 해결                                                                                |
| ----------------------------- | --------------------------- | --------------------------------------------------------------------------------- |
| DAG 안 보임                      | dags 폴더 마운트 안 됨             | docker-compose.yml 볼륨 확인                                                          |
| ModuleNotFoundError: psycopg2 | 패키지 없음                      | `docker exec -u root -it hospital-airflow pip install psycopg2-binary`            |
| API 에러                        | API 키 없음                    | `.env` 확인 + `docker cp .env hospital-airflow:/opt/airflow/.env`                   |
| er_hospitals 0건               | API 응답 없음 또는 ON CONFLICT 오류 | 로그 확인 + API 직접 호출 테스트                                                             |
| er_hourly_stats 0건            | er_realtime 데이터 없음          | Producer / Consumer 먼저 실행 후 재시도                                                   |
| JOIN 결과 없음                    | er_hospitals 미적재            | hospital_info_dag 먼저 실행                                                           |
| Worker Lost                   | Spark 리소스 부족                | docker-compose.yml `SPARK_WORKER_MEMORY=2G` + spark-submit `--executor-memory 1g` |

>Docker Compose 메모리 설정 상세 → [[Docker_Compose_Setup#리소스 설정 — Spark Worker Lost 에러]] 참고

✅ 완료되면 → [[05_Hospital_Superset]] 으로 이동 (지역별 차트 / 지도 추가)