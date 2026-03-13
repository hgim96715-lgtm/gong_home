---
aliases:
  - Superset 대시보드
  - 열차 대시보드
  - Apache Superset
  - Superset PostgreSQL 연결
  - Superset 설치
tags:
  - Project
related:
  - "[[00_Seoul Station Real-Time Train Project|train-project]]"
  - "[[01_Docker_Setup_Postgresql_Setup]]"
  - "[[04_Spark_Streaming]]"
  - "[[Superset_Setup]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_Regular_Expression]]"
  - "[[SQL_NULL_Functions]]"
  - "[[SQL_Numeric_Functions]]"
---

# 05_Superset_Dashboard — 데이터 시각화

## 이 단계의 목표

```
STEP 4 에서 PostgreSQL 에 쌓인 데이터를
Apache Superset 으로 시각화하는 것

train_schedule  → 오늘 운행 열차 수 / 노선별 분포
train_realtime  → 현재 운행 중 열차 / 진행률 분포
train_delay     → 지연 현황 / 노선별 평균 지연
```

---

---

# ① Docker Compose 에 Superset 추가

> [[01_Docker_Setup_Postgresql_Setup]] 의 `docker-compose.yml` 에 아래 서비스를 추가한다.

```yaml
# docker-compose.yml 에 추가
  superset:
    image: apache/superset:3.1.0
    container_name: train-superset
    ports:
      - "8088:8088"          # Superset Web UI
    env_file:
      - .env                 # SUPERSET_SECRET_KEY 포함
    environment:
      - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
    volumes:
      - superset_data:/app/superset_home
    networks:
      - train-network
    depends_on:
      - postgres
```

```yaml
# volumes 블록에 추가
volumes:
  kafka_data:
  postgres_data:
  superset_data:        # ← 추가
```

## SUPERSET_SECRET_KEY 생성 및 .env 등록

```
SUPERSET_SECRET_KEY:
  Superset 세션 암호화에 사용하는 비밀키
  외부에 노출되면 세션 위조 가능 → .env 에 넣고 git 에 올리지 않음
  하드코딩 금지 → ${SUPERSET_SECRET_KEY} 로 .env 에서 주입
```

```bash
# 터미널에서 키 생성
openssl rand -base64 42
# 출력 예시: kJ3mN8pQ2rT5vX7yZ0aB1cD4eF6gH9iL+Mw==

# 생성된 값을 .env 에 붙여넣기
SUPERSET_SECRET_KEY=kJ3mN8pQ2rT5vX7yZ0aB1cD4eF6gH9iL+Mw==
```


```bash
# .env 최종 형태
POSTGRES_USER=train_user
POSTGRES_PASSWORD=train_password
POSTGRES_DB=train_db
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
SUPERSET_SECRET_KEY=여기에_openssl_생성값_붙여넣기
```

```
openssl rand -base64 42 원리:
  rand       → 무작위 바이트 생성
  -base64    → 바이너리를 base64 문자열로 인코딩 (복붙 가능한 형태)
  42         → 42바이트 → 56자 문자열 생성
              숫자가 클수록 키가 길고 안전
```

```
Superset 이미지:
  apache/superset:3.1.0 공식 이미지 사용
  bitnami/superset 도 있으나 apache 공식 권장
```

---

---

# ② Superset 초기 설정

```
컨테이너를 올린 것만으로는 바로 사용 불가
최초 1회 초기화 작업 필요
```

## 초기화 순서

```bash
# 1. 컨테이너 실행
docker compose up -d

# 2. DB 초기화 (Superset 내부 메타데이터 DB 생성)
docker exec -it train-superset superset db upgrade

# 3. 관리자 계정 생성
docker exec -it train-superset superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname Admin \
  --email admin@localhost \
  --password admin

# 4. 기본 역할 / 권한 초기화
docker exec -it train-superset superset init
```

```
순서 중요:
  db upgrade → fab create-admin → init 순서 지켜야 함
  init 먼저 하면 에러 발생 가능

초기화 완료 후 접속:
  http://localhost:8088
  ID: admin / PW: admin
```

---

---

# ③ PostgreSQL 데이터 소스 연결

```
Settings → Database Connections → + Database
```

## 연결 정보

```
Database type: PostgreSQL

SQLAlchemy URI:
  postgresql://train_user:train_password@postgres:5432/train_db

⚠️ localhost:5433 ❌ — Superset 도 컨테이너 안에서 실행
   postgres:5432 ✅  — Docker 서비스명:내부 포트 사용
```

```
URI 형식:
  postgresql://{USER}:{PASSWORD}@{HOST}:{PORT}/{DB}
  postgresql://train_user:train_password@postgres:5432/train_db
```

## 연결 확인

```
Test Connection → "Connection looks good!" 메시지 확인 후 저장
```

---

---

# ④ 데이터셋 등록

```
SQL Lab → SQL Editor 또는
Data → Datasets → + Dataset

연결한 DB → public 스키마 → 테이블 선택
```

## 등록할 테이블 3개

```
train_schedule   운행계획 (오늘 열차 목록)
train_realtime   실시간 현황 (운행 중 / 진행률)
train_delay      지연 분석 (전날 지연 통계)
```

---

---

# ⑤ 차트 구성(내가 궁금한걸로 구성)

## 차트 1 — 도착역별 열차 수 (train_schedule)

```
"오늘날짜이면서 서울에서 출발하는 열차중  어디로 가장 많이 가나?"


Chart type: Pie Chart
Dataset:    train_schedule
Dimension:  arvl_stn_nm   (도착역 이름)
Metric:     COUNT(*)
Filter:     run_ymd = TO_CHAR(NOW(), 'YYYYMMDD')   ← 오늘 날짜
            dptre_stn_nm = '서울'                   ← 서울 출발만
```

```sql
-- SQL Lab 에서 미리 확인
SELECT arvl_stn_nm as "도착역",COUNT(*) as "도착역 하루 횟수"
FROM train_schedule 
WHERE dptre_stn_nm='서울' AND run_ymd::DATE = CURRENT_DATE
GROUP BY "도착역"
ORDER BY "도착역 하루 횟수" DESC;
```

## 차트 2 —  현재 운행 현황 표 (train_realtime)  -최대한 기차 전광판이랑 비슷하게 

```
"지금 운행 중인 열차들 한눈에 보기"

문제:
  같은 열차가 60초마다 계속 쌓임 → 중복 행 존재
  status 에 진행률이 섞여 있어 표로 보기 지저분함
  자정 통과 열차 진행률 오계산 (23:50 출발 → 00:10 도착)
  → 가상 Dataset + DISTINCT ON + 자정 보정 공식으로 해결
```

```sql
WITH LatestTrains AS (
    SELECT DISTINCT ON (trn_no)
        trn_no       AS "열차번호",
        plan_dep     AS "출발 시간",
        arvl_stn_nm  AS "도착역",
        status       AS "원본_상태",
        CASE
            WHEN status LIKE '운행 중%' THEN SUBSTRING(status FROM '약 (.+)\s*후 도착예정') || '후 도착예정'
            WHEN status LIKE '출발%'   THEN SUBSTRING(status FROM '(출발 (?:[0-9]+시간\s*)?(?:[0-9]+분)?전)') || ' (대기중)'
            WHEN status LIKE '%탑승 마감%' THEN '곧 출발 (탑승 마감)'
            WHEN status LIKE '곧%'     THEN SUBSTRING(status FROM '([0-9]+분\s*후)') || ' 출발 (탑승중)'
        END AS "운행 상태",

        -- 1. 현재 KST 시각과 출발/도착 시각의 초(Seconds) 차이 계산
        EXTRACT(EPOCH FROM ((NOW() AT TIME ZONE 'Asia/Seoul')::TIME - plan_dep::TIME)) AS raw_elapsed,
        EXTRACT(EPOCH FROM (plan_arr::TIME - plan_dep::TIME))                          AS raw_total

    FROM train_realtime
    WHERE plan_dep != '--:--' AND plan_arr != '--:--'   -- 에러 데이터 방어
    ORDER BY trn_no, created_at DESC
),
ProgressCalc AS (
    SELECT
        *,
        -- 2. 자정(Midnight) 랩어라운드 보정 (+24시간 = 86400초)
        -- 현재시간 - 출발시간이 -12시간 이하면 자정을 지난 것 → +24시간
        CASE WHEN raw_elapsed < -43200 THEN raw_elapsed + 86400 ELSE raw_elapsed END AS elapsed_sec,
        -- 도착시간 < 출발시간이면 자정 통과 열차 → +24시간
        CASE WHEN raw_total   < 0      THEN raw_total   + 86400 ELSE raw_total   END AS total_sec
    FROM LatestTrains
),
FinalCalc AS (
    SELECT
        *,
        -- 3. 최종 퍼센트 계산 (0~100 고정)
        LEAST(GREATEST(ROUND(
            (elapsed_sec / NULLIF(total_sec, 0)) * 100
        ), 0), 100) AS pct
    FROM ProgressCalc
)
SELECT
    "열차번호",
    "출발 시간",
    "도착역",
    "운행 상태",
    CASE
        WHEN "원본_상태" LIKE '운행 중%'
        THEN '진행률: ' || repeat('█', ceil(pct/10.0)::int)
                        || repeat('░', 10-ceil(pct/10.0)::int)
                        || ' ' || pct || '%'
        ELSE '-'
    END AS "진행률"
FROM FinalCalc
WHERE pct < 100;    -- 도착 완료 열차 제외
```

```text
CTE 3단계 구조:
  LatestTrains   DISTINCT ON → 열차별 최신 1행 + raw 초 계산
  ProgressCalc   자정 보정   → elapsed_sec / total_sec 확정
  FinalCalc      진행률 %    → 0~100 클램핑

자정 보정 원리:
  86400 = 하루 전체 초 (24 × 60 × 60)
  43200 = 반나절 초    (12 × 60 × 60)

  raw_elapsed < -43200  → 현재시각이 출발보다 9시간 이상 앞선 것처럼 보임
                          = 자정 넘어서 생긴 음수 → +86400
  raw_total < 0         → 도착시각 < 출발시각 = 자정 통과 열차 → +86400

  예) 23:50 출발 → 00:30 도착
      raw_total = 00:30 - 23:50 = -84600초 → +86400 = 1800초(30분) ✅

진행률 계산 — progress_pct 저장값 대신 NOW() 실시간:
  ❌ 저장값 → Producer 멈추면 마지막 값에서 고정
  ✅ NOW() AT TIME ZONE 'Asia/Seoul' → 항상 현재 KST 기준

NULLIF(total_sec, 0):
  plan_dep = plan_arr 인 경우 0 나눗셈 방지

WHERE pct < 100:
  도착 완료 열차 제외 (전광판에 불필요)
```

  → [[SQL_Regular_Expression]] 참고

### 차트 설정

```text
Chart type: Table
Dataset:    위 가상 Dataset 으로 등록
Columns:    열차번호 / 출발 시간 / 도착역 / 운행 상태 / 진행률
Sort:       출발 시간 ASC
```

### 결과 예시

|열차번호|노선|출발 시간|도착역|운행 상태|진행률(%)|
|---|---|---|---|---|---|
|00065|경부선|20:28|부산|2시간 17분 후 도착예정|진행률: █░░░░░░░░░ 10%|
|00109|경부선|20:49|부산|2시간 58분 후 도착예정|진행률: █░░░░░░░░░ 8%|
|00067|경부선|20:58|부산|2시간 32분 후 도착예정|진행률: █░░░░░░░░░ 8%|
|00069|경부선|21:28|부산|2시간 34분 후 도착예정|진행률: █░░░░░░░░░ 1%|
|00111|경부선|21:38|부산|7분 후 출발(탑승중)|-|
|00071|경부선|21:58|부산|출발 27분전 (대기중)|-|
|00073|경부선|22:28|부산|출발 57분전 (대기중)|-|
|00117|경부선|22:58|동대구|출발 1시간 27분전 (대기중)|-|

 

## 차트 3 — 노선별  도착  평균 지연 (train_delay)

```
Chart type: Bar Chart
Dataset:    train_delay
Y-axis:     mrnt_nm
Y-Axis Sort By : Average value
Metric:     AVG(출발지연), AVG(도착지연)
Filter:     도착지연 > 0
```

>도착지연 > 0 필터만 걸어두면, **"결과적으로 지연된 열차들이, 출발할 때는 얼마나 늦었길래 도착도 늦은 걸까?"** 라는 흐름으로 원인을 파악하기 훨씬 좋습니다.

```sql
SELECT mrnt_nm AS mrnt_nm,
       ROUND(AVG("출발지연"),2) AS "AVG(출발지연)",
       ROUND(AVG("도착지연"),2) AS "AVG(도착지연)"
FROM
  (SELECT mrnt_nm,
          dep_delay::int as "출발지연",
          arr_delay::int as "도착지연",
          SPLIT_PART(arr_status, '(', 1) as "도착 상태",
          SPLIT_PART(dep_status, '(', 1) as "출발 상태"
   FROM train_delay
   WHERE arr_delay >0) AS virtual_table
WHERE "도착지연" > 0
GROUP BY mrnt_nm
ORDER BY "AVG(출발지연)" DESC
```

## 차트 4 — 시간별 기차 이동 횟수 (train_schedule)

```text
"언제 기차가 가장 많이 이동할까?"

Chart type: Line Chart
Dataset:    train_schedule (가상 Dataset)
X-axis:     hour (시간대)
Metric:     COUNT(*)
```

```text
⚠️ trn_plan_dptre_dt 는 VARCHAR
   EXTRACT 하려면 반드시 ::TIMESTAMP 캐스팅 먼저
   EXTRACT(HOUR FROM trn_plan_dptre_dt)         ❌
   EXTRACT(HOUR FROM trn_plan_dptre_dt::TIMESTAMP) ✅
```

## 차트 5 — 어제 운행횟수 vs 오늘 운행횟수 (상단 Table)

```text
"어제랑 오늘 운행 편수가 같은가? 요일/날씨에 따라 달라지나?"

Chart type: Table
Dataset:    train_schedule (가상 Dataset)
Columns:    날짜 기준 / 운행 횟수
```

```sql
SELECT
    CASE
        WHEN trn_plan_dptre_dt::DATE = CURRENT_DATE - INTERVAL '1 day'
        THEN '어제 [' || TO_CHAR(CURRENT_DATE - INTERVAL '1 day', 'YYYY-MM-DD') || ']'
        WHEN trn_plan_dptre_dt::DATE = CURRENT_DATE
        THEN '오늘 [' || TO_CHAR(CURRENT_DATE, 'YYYY-MM-DD') || ']'
    END AS "날짜 기준",
    COUNT(trn_no) AS "운행 횟수"
FROM train_schedule
-- INTERVAL 계산 결과를 ::DATE 로 명확히 맞춰주면 에러 날 확률 0%
WHERE trn_plan_dptre_dt::DATE IN (CURRENT_DATE, (CURRENT_DATE - INTERVAL '1 day')::DATE)
GROUP BY trn_plan_dptre_dt::DATE
ORDER BY "날짜 기준" ASC;
```

```text
포인트:
  CASE WHEN 으로 날짜를 "어제 [2026-03-12]" 형태 레이블로 변환
  IN (CURRENT_DATE, ...) 으로 어제·오늘 두 날짜만 필터
  ::DATE 캐스팅 통일 → INTERVAL 계산 결과 타입 불일치 에러 방지
```

> 나중에 어제 오늘 말고 일주일 치를 비교 해봐야하나? 

## 차트 6 — Big Number 카드 3개 (상단)

```text
"한눈에 보이는 오늘의 핵심 숫자"
```

| 카드          | 타입         | Dataset        | SQL / Metric                                               |
| ----------- | ---------- | -------------- | ---------------------------------------------------------- |
| 오늘 열차 운행횟수  | Big Number | train_schedule | COUNT(*) WHERE run_ymd = 오늘                                |
| 전날 기준 지연 횟수 | Big Number | train_delay    | COUNT(*) WHERE arr_delay > 0                               |
| 지금 곧출발하는 열차 | Table      | train_realtime | DISTINCT ON WHERE status LIKE '곧%' or status LIKE '%탑승마감%' |

```sql
-- 오늘 열차 운행횟수
SELECT COUNT(*) FROM train_schedule
WHERE dptre_stn_nm = '서울'
  AND run_ymd::DATE = CURRENT_DATE;

-- 전날 기준 지연 횟수
SELECT COUNT(*) FROM train_delay
WHERE arr_delay > 0;

-- 지금 곧출발하는 열차
WITH LatestTrainStatus AS (
    SELECT DISTINCT ON (trn_no)
        trn_no, plan_dep, arvl_stn_nm, status, progress_pct
    FROM train_realtime
    ORDER BY trn_no, created_at DESC
),
SoonTrains AS (
    SELECT
        trn_no       AS "열차번호",
        plan_dep     AS "출발 시간",
        arvl_stn_nm  AS "도착역",
        CASE
            WHEN status LIKE '운행 중%'    THEN SUBSTRING(status FROM '약 (.+)\s*후 도착예정') || '후 도착예정'
            WHEN status LIKE '출발%'       THEN SUBSTRING(status FROM '(출발 (?:[0-9]+시간\s*)?(?:[0-9]+분)?전)') || ' (대기중)'
            WHEN status LIKE '%탑승 마감%' THEN '곧 출발 (탑승 마감)'
            WHEN status LIKE '곧%'         THEN SUBSTRING(status FROM '([0-9]+분\s*후)') || ' 출발 (탑승중)'
        END AS "운행 상태",
        CASE
            WHEN status LIKE '운행 중%'
            THEN '진행률: ' || repeat('█', ceil(progress_pct/10.0)::int)
                             || repeat('░', 10-ceil(progress_pct/10.0)::int)
                             || ' ' || progress_pct || '%'
            ELSE '-'
        END AS "진행률"
    FROM LatestTrainStatus
    WHERE status LIKE '곧%' OR status LIKE '%탑승 마감%'
)
SELECT * FROM SoonTrains
UNION ALL
-- 곧 출발 열차가 없을 때 안내 메시지 표시
SELECT
    '-' AS "열차번호",
    '-' AS "출발 시간",
    '-' AS "도착역",
    '현재 곧 출발하는 열차가 없습니다.' AS "운행 상태",
    '-' AS "진행률"
WHERE NOT EXISTS (SELECT 1 FROM SoonTrains);
```


---

---

# ⑥ 대시보드 구성

```
Dashboards → + Dashboard → 이름: 서울역 출발 기준으로 기차는 어디로 흘러갈까 🚆

자동 새로고침:
  Dashboard → ··· → Set auto-refresh → 30 seconds
  → train_realtime 데이터 자동 반영
```


---
---

# 결과 Dashboard

>http://localhost:8088/superset/dashboard/p/0J7mJeKyAGB/


![[서울역-출발-기준으로-기차는-어디로-흘러갈까-🚆-2026-03-13T10-10-28.372Z.jpg]]


```text
오늘 열차 운행횟수: 789
가장 많이 도착하는 곳: 부산 74회
전날 기준 지연 횟수: 91
경전선 도착 평균 지연 최대 (~6분)
부산행 열차가 압도적으로 많음 (Pie 에서 가장 큰 비중)
출퇴근 시간대(6시 운행횟수 65회, 18시 56회) 운행 횟수 피크 확인, 
오전 11시에 운행횟수 27 -> 기차 이동이 적음(왜그럴까? 점심시간인가?)
왜 15시 57회일까?
```

>**주간 안전 점검**을 하는 황금 시간대가 바로 오전 10시~12시 사이라고 한다

---
## 차트 목록

| 위치   | 차트 제목                       | 타입                | 데이터                                                           |
| ---- | --------------------------- | ----------------- | ------------------------------------------------------------- |
| 상단   | 어제운행 횟수 vs 오늘 운행 횟수         | Table             | train_schedule 날짜 비교                                          |
| 상단   | 오늘 열차 운행횟수                  | Big Number        | train_schedule COUNT(*)                                       |
| 상단   | 전날 기준 지연 횟수                 | Big Number        | train_delay COUNT(*) WHERE arr_delay > 0                      |
| 상단   | 지금 곧출발하는 열차                 | Table             | train_realtime WHERE status LIKE '곧%' or status LIKE %탑승 마감%' |
| 중단 좌 | 오늘의 열차들은 어디로 많이갈까?🤔        | Pie Chart         | train_schedule                                                |
| 중단 우 | 실시간 기차 전광판 🚉               | Table             | train_realtime DISTINCT ON + NOW() KST                        |
| 하단 좌 | 시간별 언제 기차가 많이 이동할까?         | Line Chart        | train_schedule by hour                                        |
| 하단 우 | 기차 운행 패턴 분석 📊              | Text (Markdown)   | 관찰 포인트 + 원인 추정                                                |
| 최하단  | 어제 어디 노선이 많이 지연되었을까?!(전날통계) | Grouped Bar Chart | train_delay AVG(dep_delay), AVG(arr_delay)                    |



---


# ⑦ SQL Lab — 데이터 직접 확인

```
SQL Lab → SQL Editor → train_db 선택
```

```sql
-- 어제 vs 오늘 운행횟수 비교
SELECT
    CASE
        WHEN trn_plan_dptre_dt::DATE = CURRENT_DATE - INTERVAL '1 day'
        THEN '어제 [' || TO_CHAR(CURRENT_DATE - INTERVAL '1 day', 'YYYY-MM-DD') || ']'
        WHEN trn_plan_dptre_dt::DATE = CURRENT_DATE
        THEN '오늘 [' || TO_CHAR(CURRENT_DATE, 'YYYY-MM-DD') || ']'
    END AS "날짜 기준",
    COUNT(trn_no) AS "운행 횟수"
FROM train_schedule
WHERE trn_plan_dptre_dt::DATE IN (CURRENT_DATE, (CURRENT_DATE - INTERVAL '1 day')::DATE)
GROUP BY trn_plan_dptre_dt::DATE
ORDER BY "날짜 기준" ASC;

-- 현재 운행 중인 열차 (실시간 기차 전광판) — 자정 보정 포함
WITH LatestTrains AS (
    SELECT DISTINCT ON (trn_no)
        trn_no, plan_dep, plan_arr, arvl_stn_nm, status,
        CASE
            WHEN status LIKE '운행 중%'    THEN SUBSTRING(status FROM '약 (.+)\s*후 도착예정') || '후 도착예정'
            WHEN status LIKE '출발%'       THEN SUBSTRING(status FROM '(출발 (?:[0-9]+시간\s*)?(?:[0-9]+분)?전)') || ' (대기중)'
            WHEN status LIKE '%탑승 마감%' THEN '곧 출발 (탑승 마감)'
            WHEN status LIKE '곧%'         THEN SUBSTRING(status FROM '([0-9]+분\s*후)') || ' 출발 (탑승중)'
        END AS "운행 상태",
        EXTRACT(EPOCH FROM ((NOW() AT TIME ZONE 'Asia/Seoul')::TIME - plan_dep::TIME)) AS raw_elapsed,
        EXTRACT(EPOCH FROM (plan_arr::TIME - plan_dep::TIME))                          AS raw_total
    FROM train_realtime
    WHERE plan_dep != '--:--' AND plan_arr != '--:--'
    ORDER BY trn_no, created_at DESC
),
ProgressCalc AS (
    SELECT *,
        CASE WHEN raw_elapsed < -43200 THEN raw_elapsed + 86400 ELSE raw_elapsed END AS elapsed_sec,
        CASE WHEN raw_total   < 0      THEN raw_total   + 86400 ELSE raw_total   END AS total_sec
    FROM LatestTrains
),
FinalCalc AS (
    SELECT *,
        LEAST(GREATEST(ROUND((elapsed_sec / NULLIF(total_sec, 0)) * 100), 0), 100) AS pct
    FROM ProgressCalc
)
SELECT
    trn_no AS "열차번호", plan_dep AS "출발 시간", arvl_stn_nm AS "도착역",
    "운행 상태",
    CASE WHEN status LIKE '운행 중%'
        THEN '진행률: ' || repeat('█', ceil(pct/10.0)::int)
                        || repeat('░', 10-ceil(pct/10.0)::int)
                        || ' ' || pct || '%'
        ELSE '-'
    END AS "진행률"
FROM FinalCalc
WHERE pct < 100;

-- 오늘 서울 출발 열차 어디로 많이 가나?
SELECT arvl_stn_nm AS "도착역", COUNT(*) AS "도착역 하루 횟수"
FROM train_schedule
WHERE dptre_stn_nm = '서울'
  AND run_ymd::DATE = CURRENT_DATE
GROUP BY "도착역"
ORDER BY "도착역 하루 횟수" DESC;

-- 전날 지연 현황
SELECT
    mrnt_nm AS "노선명",
    dep_delay::int AS "출발지연",
    arr_delay::int AS "도착지연",
    SPLIT_PART(arr_status, '(', 1) AS "도착 상태",
    SPLIT_PART(dep_status, '(', 1) AS "출발 상태"
FROM train_delay
WHERE arr_delay > 0;

-- 시간별 기차 이동 횟수 [[SQL_Date_Functions#⚠️ VARCHAR 컬럼에는 EXTRACT 바로 못 씀 — TIMESTAMP 필수]] 참고 
SELECT
    trn_plan_dptre_dt::TIMESTAMP AS "Hour"  -- ⚠️ TIMESTAMP 로 캐스팅 필수
FROM train_schedule
ORDER BY "Hour" ASC;
-- EXTRACT(HOUR FROM ...) 쓸 때도 반드시 ::TIMESTAMP 먼저
-- EXTRACT(HOUR FROM trn_plan_dptre_dt::TIMESTAMP) AS hour
```


---

---

# 트러블슈팅....에러 안만날래.. 🤦‍♀️

|증상|원인|해결|
|---|---|---|
|`superset db upgrade` 후 접속 불가|`fab create-admin` 또는 `superset init` 누락|초기화 3단계 순서대로 실행|
|`Connection refused` (PostgreSQL)|URI 에 localhost:5433 사용|`postgres:5432` 로 수정|
|테이블이 데이터셋에 안 보임|스키마 미선택|Dataset 등록 시 스키마 `public` 선택|
|차트 데이터 없음|Producer / Consumer 미실행|`docker compose ps` 로 상태 확인|
|자동 새로고침 후 데이터 안 바뀜|캐시 활성화|Dashboard → Edit → Cache timeout = 0|
|`SUPERSET_SECRET_KEY` 경고|기본값 사용 중|`.env` 에 고유한 키 값 설정|
|`EXTRACT(HOUR FROM 컬럼)` 결과 이상|VARCHAR 컬럼에 바로 EXTRACT 적용|`컬럼::TIMESTAMP` 로 캐스팅 후 EXTRACT|
|진행률이 전부 0%|컨테이너 내부 시각이 UTC → plan_dep(KST) 와 차이 발생|`NOW()::TIME` 대신 `(NOW() AT TIME ZONE 'Asia/Seoul')::TIME` 사용|
|자정 통과 열차 진행률 오계산|23:50 출발 → 00:30 도착 시 raw_total 이 음수|`CASE WHEN raw_total < 0 THEN raw_total + 86400` 으로 보정|
|**차트 숫자가 2배·3배로 뻥튀기**|**Producer 재실행 시 동일 데이터 중복 발행**|**`producer_state.json` 도입 → 아래 참고**|

## Producer 재실행 시 Superset 데이터 중복 문제

```
증상: Producer 를 껐다 켜면 차트 수치가 2배, 3배로 증가
원인: Producer 가 재실행될 때마다 run_schedule(), run_delay_analysis() 가
      이미 발행한 데이터를 다시 Kafka 로 전송
      → Spark → DB 파이프라인이 그대로 중복 적재
```

**해결: `producer_state.json` 상태 파일로 발행 이력 관리**


```json
{
  "last_schedule_date": "20260313",
  "last_delay_analysis_date": "20260312"
}
```

## ⚠️ `.gitignore` 에 반드시 추가

```gitignore
# 상태 저장 파일 무시
producer_state.json
```

```
동작 원리:
  Producer 최초 실행  → 파일 없음 → 정상 발행 → 파일 생성
  Producer 재실행     → 파일 읽음 → 오늘 날짜와 비교 → 같으면 스킵
  다음날 이후         → 날짜 바뀜 → 정상 발행

  ⚠️ DB 초기화 후 처음부터 다시 받으려면
     producer/producer_state.json 삭제 후 재실행
```

>상세 코드 → [[03_Kafka_Producer#실제 겪은 트러블슈팅#8. Producer 재실행 시 Superset 데이터 중복 문제|트러블 슈팅-8]]  참고

---

---

✅ 완료되면 → [[06_Airflow_Pipeline]] 으로 이동