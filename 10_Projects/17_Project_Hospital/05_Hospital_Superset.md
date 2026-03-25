---
aliases:
  - superset
  - Hospital Superset
  - 응급실 대시보드
  - 병상 현황 차트
tags:
  - Project
related:
  - "[[00_Hospital_Project]]"
  - "[[Superset_Setup]]"
  - "[[PostgreSQL_Setup]]"
---
# 05_Hospital_Superset — 응급실 대시보드

## 이 단계의 목표

```
① Superset 초기 세팅 + PostgreSQL 연결
② Dataset 등록 (er_realtime / JOIN 쿼리)
③ 차트 8개 구성
④ 지도 차트 (위도/경도 활용)
⑤ 대시보드 레이아웃 완성 + 자동 새로고침

데이터 품질 고려사항:
  region 대부분 서울 → 필터 확인 필요
  mk 컬럼 NULL 많음 → IS NOT NULL 필터 처리
  hvec 음수 가능 → 포화/수용 구분 필요
```

---

---

# ① Superset 초기 세팅

```bash
# 컨테이너 실행 확인
docker compose ps

# 브라우저 접속
http://localhost:8089
# ID: admin / PW: admin
```

> 초기화 먼저 → [[Superset_Setup#③ 초기화 순서 — Docker]] 참고

---

---

# ② PostgreSQL 연결

```
Settings → Database Connections → + Database → PostgreSQL

HOST:     postgres        ← 서비스명 (컨테이너 내부)
PORT:     5432            ← 내부 포트
DATABASE: hospital_db
USERNAME: hospital_user
PASSWORD: hospital_password
```

```
⚠️ localhost:5434 로 연결하면 안 됨
  Superset 도 Docker 컨테이너 안 → 서비스명:내부포트 사용
  localhost:5434  → 맥북 터미널 / DataGrip
  postgres:5432   → 컨테이너 안 (Superset / Spark / Airflow)
```

---

---

# ③ Dataset 등록

## 기본 Dataset — er_realtime

```
Datasets → + Dataset
Database: hospital_db
Schema:   public
Table:    er_realtime
```

## JOIN Dataset — er_realtime + er_hospitals 

```
Datasets → + Dataset → CREATE DATASET FROM SQL
```

```sql
-- 최신 시각 기준 실시간 병상 + 병원 기본정보 JOIN
SELECT
    r.hpid,
    r.hpname,
    r.hvec,
    r.hvoc,
    r.hvgc,
    r.hvctayn,
    r.hvventiayn,
    r.hvmriayn,
    r.hvangioayn,
    r.notice_msg,
    r.created_at,
    h.region,
    h.duty_addr,
    h.duty_tel,
    h.duty_eryn,
    h.wgs84_lat,
    h.wgs84_lon,
    h.hpbdn,
    h.mk_stroke,
    h.mk_cardiac,
    h.mk_trauma,
    h.mk_pediatric
FROM er_realtime r
JOIN er_hospitals h ON r.hpid = h.hpid
WHERE r.created_at = (
    SELECT MAX(created_at) FROM er_realtime
)
```

```
⚠️ er_hospitals 가 서울만 수집되어 JOIN 해도 region 이 서울만 나옴
→ 전국 지역 분석은 아래 duty_tel 패턴 사용
```

## 지역 추정 Dataset — duty_tel 지역번호 활용 ⭐️

```
er_realtime 에는 전국 병원이 들어오지만 지역 정보가 없음
er_hospitals JOIN 해도 서울만 나옴 (서울만 수집됐기 때문)

뭘로 해야할지 고민하다가 지역번호로 하면 되겠다고 생각하여 지역번호로 추정해보았다.
→ duty_tel 앞 지역번호로 지역 추정
   02 → 서울 / 051 → 부산 / 053 → 대구 ...
→ er_realtime 단독으로 전국 지역별 분석 가능
```

```sql
-- er_realtime 단독 + 지역번호로 지역 추정
SELECT
    r.hpid,
    r.hpname,
    r.hvec,
    r.created_at,
    CASE
        WHEN r.duty_tel LIKE '02%'  THEN '서울'
        WHEN r.duty_tel LIKE '031%' THEN '경기'
        WHEN r.duty_tel LIKE '032%' THEN '인천'
        WHEN r.duty_tel LIKE '033%' THEN '강원'
        WHEN r.duty_tel LIKE '041%' THEN '충남'
        WHEN r.duty_tel LIKE '042%' THEN '대전'
        WHEN r.duty_tel LIKE '043%' THEN '충북'
        WHEN r.duty_tel LIKE '044%' THEN '세종'
        WHEN r.duty_tel LIKE '051%' THEN '부산'
        WHEN r.duty_tel LIKE '052%' THEN '울산'
        WHEN r.duty_tel LIKE '053%' THEN '대구'
        WHEN r.duty_tel LIKE '054%' THEN '경북'
        WHEN r.duty_tel LIKE '055%' THEN '경남'
        WHEN r.duty_tel LIKE '061%' THEN '전남'
        WHEN r.duty_tel LIKE '062%' THEN '광주'
        WHEN r.duty_tel LIKE '063%' THEN '전북'
        WHEN r.duty_tel LIKE '064%' THEN '제주'
        ELSE '기타'
    END AS region_from_tel
FROM er_realtime r
WHERE r.created_at = (SELECT MAX(created_at) FROM er_realtime)
  AND r.duty_tel IS NOT NULL
```

```sql
-- 지역별 집계 버전
SELECT
    CASE
        WHEN duty_tel LIKE '02%'  THEN '서울'
        WHEN duty_tel LIKE '031%' THEN '경기'
        WHEN duty_tel LIKE '051%' THEN '부산'
        WHEN duty_tel LIKE '053%' THEN '대구'
        -- ...
        ELSE '기타'
    END AS region,
    COUNT(*) AS hospital_count,
    ROUND(AVG(hvec), 1) AS avg_beds,
    COUNT(*) FILTER(WHERE hvec <= 0) AS saturated
FROM er_realtime
WHERE created_at = (SELECT MAX(created_at) FROM er_realtime)
  AND duty_tel IS NOT NULL
GROUP BY region
ORDER BY hospital_count DESC;
```

```
장점:
  er_realtime 단독 → er_hospitals JOIN 불필요
  전국 데이터에 바로 적용 가능
  duty_tel 없는 병원만 '기타' 로 처리

주의:
  duty_tel 이 NULL 이거나 형식 다른 경우 → '기타' 처리 > 기타 처리는 10 미만 이라서 다행이었다.. 그래도 차이가 있을 것 같다.
  휴대폰 번호(010/011 등) 등록된 병원은 지역 특정 불가
```

## 시간대별 지역별 집계 Dataset ⭐️

```
er_hourly_status  → 서울 전용 / DAG 미리 집계 (빠름)
이 Dataset       → 전국 지역별 / er_realtime 직접 집계

용도:
  서울 시간대별 상세 분석  → er_hourly_status
  전국 지역별 비교        → 이 Dataset
```

```sql
-- Superset 가상 Dataset — 시간대별 지역별 집계
SELECT
    region,
    DATE_TRUNC('hour', created_at)      AS stat_hour,
    TO_CHAR(created_at, 'HH24') || '시' AS 시간대,
    ROUND(AVG(hvec), 1)                 AS avg_beds,
    COUNT(*) FILTER(WHERE hvec <= 0)    AS zero_count,
    COUNT(*)                            AS total_hospitals,
    ROUND(
        COUNT(*) FILTER(WHERE hvec <= 0)::NUMERIC
        / NULLIF(COUNT(*), 0) * 100
    , 1)                                AS saturation_pct
FROM er_realtime
WHERE created_at >= (NOW() AT TIME ZONE 'Asia/Seoul')::DATE
  AND region != '기타'
GROUP BY region, DATE_TRUNC('hour', created_at)
ORDER BY region, stat_hour;
```

```
Superset 차트 구성:
  Chart type: Mixed Chart (Line + Bar)
  X축:        시간대
  Bar:        avg_beds
  Line:       saturation_pct
  Filter:     region 선택 (서울 / 부산 / 대구 등)
```

---

---

# ④ 차트 구성

## 차트 1 — 병원별 실시간 병상 수

```
Chart type: Bar Chart (수평)
Dataset:    er_realtime

Metrics:    MAX(hvec)
Dimensions: hpname
Sort:       MAX(hvec) DESC
Row limit:  30

Filters:
  hvec > 0   ← 음수(포화) 제외 / 수용 가능 병원만
```

```sql
SELECT hpname, hvec
FROM er_realtime
WHERE created_at = (SELECT MAX(created_at) FROM er_realtime)
  AND hvec > 0
ORDER BY hvec DESC
LIMIT 30;
```

---

## 차트 2 — 병상 포화 병원 현황

```
Chart type: Table
Dataset:    er_realtime

Columns:    hpname / hvec / notice_msg / created_at
Filters:    hvec <= 0
Sort:       hvec ASC   ← 가장 부족한 병원 위로
```

```
hvec 값 해석:
  hvec > 0   수용 가능
  0<hvec <=10   응급실 포화 직전
  hvec < 0   응급실 포화 상태 
```


---

## 차트 4 — 유효 공지 병원 목록

```
Chart type: Table
Dataset:    er_realtime

Columns:  hpname / notice_msg / created_at
Filters:
  notice_msg IS NOT NULL
  notice_msg != ''
  notice_msg NOT LIKE '%119 구급대원%'   ← 기본 메시지 제외
```

```sql
SELECT hpname, notice_msg, created_at
FROM er_realtime
WHERE notice_msg IS NOT NULL
  AND notice_msg != ''
  AND notice_msg NOT LIKE '%119 구급대원%'
ORDER BY created_at DESC;
```

---

## 차트 5 — 병원 위치 지도 ⭐️ (위도/경도 활용)

```
Chart type: Deck.gl Scatter Plot  또는  Deck.gl Geojson
Dataset:    er_realtime + er_hospitals JOIN

Longitude:  wgs84_lon
Latitude:   wgs84_lat
Color by:   hvec   ← 병상 수에 따라 색상 변화

설정:
  Viewport: 서울 중심 (위도 37.5, 경도 127.0, Zoom 10)
  Point size: 50~100
  Opacity: 0.8
```

```
색상 의미:
  초록  hvec 많음 (수용 여유)
  노랑  hvec 적음 (부족)
  빨강  hvec < 0 (포화)

클릭 시 툴팁:
  hpname / hvec / duty_tel / region 표시
```

```sql
-- 지도 Dataset 확인용
SELECT hpname, wgs84_lat, wgs84_lon, hvec, region
FROM er_realtime r
JOIN er_hospitals h ON r.hpid = h.hpid
WHERE r.created_at = (SELECT MAX(created_at) FROM er_realtime)
  AND h.wgs84_lat IS NOT NULL
  AND h.wgs84_lon IS NOT NULL;
```

---

## 차트 6 — 지역별 평균 병상 수

```
Chart type: Bar Chart
Dataset:    JOIN Dataset

Metrics:    AVG(hvec)
Dimensions: region
Sort:       AVG(hvec) DESC

⚠️ region 대부분 서울:
  현재 수집된 병원이 서울 기반 → 지역 편중 정상
  전국 데이터 수집 시 자동 해소
  region IS NOT NULL 필터 추가 권장
```

```sql
SELECT h.region, AVG(r.hvec) AS avg_beds, COUNT(DISTINCT r.hpid) AS cnt
FROM er_realtime r
JOIN er_hospitals h ON r.hpid = h.hpid
WHERE r.hvec > 0
  AND h.region IS NOT NULL
GROUP BY h.region
ORDER BY avg_beds DESC;
```

---

## 차트 6-2 — 지역별 포화 현황 ⭐️

```
Chart type: Bar Chart (Stacked 권장)
Dataset:    er_realtime (duty_tel region 컬럼 활용)

지도 없이 지역별 포화 상태 비교
region 컬럼 = duty_tel 지역번호로 추정한 값
```

```sql
SELECT
    region,
    COUNT(*) AS total,
    COUNT(*) FILTER(WHERE hvec <= 0) AS "포화 상태",
    COUNT(*) FILTER(WHERE hvec > 0)  AS "수용 가능"
FROM er_realtime
WHERE created_at = (SELECT MAX(created_at) FROM er_realtime)
  AND region != '기타'
GROUP BY region
ORDER BY "포화 상태" DESC;
```

```
⚠️ COUNT(hvec < 0) 는 틀린 패턴
  hvec < 0 → TRUE/FALSE 반환
  COUNT(TRUE) = 1 / COUNT(FALSE) = 1
  → 포화 여부 무관하게 전체 병원 수가 나옴

✅ COUNT(*) FILTER(WHERE hvec <= 0) 사용
  조건 만족하는 행만 셈
  0이 나오면 = 그 지역에 포화 병원이 없음

포화 상태 기준:
  hvec <= 0  포화 (0 포함 / 음수 = 대기 환자 있음)
  hvec > 0   수용 가능
```

---

## 차트 7 — [서울 응급실 포화율&평균 가용병상수(응급의료기관만)]

```
Chart type: Mixed Chart (Line + Bar)
Dataset:    er_hourly_status (Airflow 집계 후)

Line:  AVG(saturation_pct)  ← 평균 포화율
Bar:   AVG(zero_count)      ← 평균 포화 병원 수
X축:   시간대

region 컬럼 제외 (현재 서울만 수집 → 의미 없음)
```

```sql
-- 시간대별 포화율 쿼리
SELECT
  DATE_TRUNC('hour', created_at) AS stat_hour,

  ROUND(
    SUM(zero_count)::NUMERIC
    / NULLIF(SUM(total_hospitals), 0) * 100
  ,1) AS "응급실 포화율"
,
  ROUND(
    SUM(avg_beds * total_hospitals)::NUMERIC
    / NULLIF(SUM(total_hospitals), 0)
  ,1) AS "병원당 평균 남아있는 병상 수"

FROM er_hourly_status
WHERE created_at >= (NOW() AT TIME ZONE 'Asia/Seoul')::DATE
GROUP BY DATE_TRUNC('hour', created_at)
ORDER BY stat_hour;

```

```
실제 확인 결과 (2시간치):
  16시  포화율 14.6%  평균 포화병원수5.5
  18시  포화율 17%    평균 남아있는 병상수 5.9

  → Consumer 계속 돌리면 시간대별 패턴 축적
  → 아침/점심/저녁/심야 비교 가능해짐
```


---

---

# ⑤ 대시보드 레이아웃

```
대시보드 제목: 🚑 응급실 실시간 병상 현황
```

![[hospital-final.jpg]]

## 자동 새로고침

```
대시보드 우상단 ... → Set auto-refresh → 30초

Producer 5분 주기 발행
Spark 30초 배치 적재
→ 30초 새로고침으로 거의 실시간 반영
```

---

---

# 데이터 품질 이슈 및 처리 방법

```region 대부분 서울:
  dutyAddr 앞 2글자 = region
  현재 수집 병원이 서울 기반 → 정상
  Superset 필터에서 region 확인
  → 전국 데이터 수집 시 자동 해소

hpbdn / mk 컬럼 NULL 많음:
  API 에서 값 없는 병원이 많음
  차트 Filters 에 IS NOT NULL 추가
  향후 COALESCE(mk_stroke, 'N') 기본값 처리 고려

mk_trauma / mk_cardiac VARCHAR(10):
  'N1' 같은 2자 이상 값 확인됨
  VARCHAR(1) → VARCHAR(10) 변경 완료
  값 의미 불명확 → 운영/미운영 구분 시 주의

hvec 음수:
  병상 초과 = 음수 (대기 환자 있음)
  차트별로 hvec > 0 / hvec <= 0 구분해서 필터

notice_msg 기본 메시지:
  대부분 병원이 동일한 기본 메시지 사용
  NOT LIKE '%119 구급대원%' 로 필터링

hvec 과 notice_msg 불일치 문제:
  hvec (병상 수)   → 자동 실시간 갱신
  notice_msg (공지) → 병원 담당자가 수동 입력

  업데이트 주기가 달라서 괴리 발생 가능:
    hvec = 5 (여유) + notice_msg = "포화 상태"
    → 병상이 채워졌지만 공지를 아직 안 바꾼 것
    hvec = 0 (포화) + notice_msg = "정상 운영 중"
    → 갑자기 환자 몰렸지만 공지 업데이트 안 됨

  결론: hvec 이 더 신뢰할 수 있는 지표
        notice_msg 는 참고용으로만 활용

er_hospitals 에 응급실 아닌 병원 포함:
  getEgytBassInfoInqire API 가 응급실 있는 병원만이 아니라
  등록된 의료기관 전체를 반환하는 경우 있음
  → duty_eryn = '1' 필터 필수 (응급실 운영 병원만)
  → 미필터 시 강남구 181개 같은 비현실적 수치 나옴

경기도 데이터 1건 섞임:
  서울 수집 중에 경기도 병원 1건이 포함됨
  → duty_addr NOT ILIKE '경기도%' 로 제외
```


---

---

# 최종 대시보드 구성 & 인사이트

## hvec 기준 상태 분류 ⭐️

```
hvec > 10        응급실 여유 있음
0 <= hvec < 10   응급실 포화 직전
hvec <= 0        응급실 포화 상태
```

> 병원 근무 경험을 바탕으로 직접 설정한 기준입니다. 응급실에서 근무하다 보면 병상이 10개 미만으로 줄어드는 순간부터 중증 환자가 한 명만 와도 바로 포화 상태로 전환되는 경우가 많았습니다. 단순히 hvec = 0 을 기준으로 삼는 것보다, 10 미만부터 "포화 직전"으로 분류하는 것이 실제 현장에 더 가까운 분류라고 판단했습니다.

## 지역 추정 방식

```
er_hospitals 는 서울만 수집됨
→ er_realtime duty_tel 지역번호로 region 추정
  02=서울 / 031=경기 / 051=부산 / 053=대구 ...
  기타(지역번호 미확인): 10건 미만 → 분석에 큰 영향 없음
```

## 차트 목록

|차트|타입|Dataset|
|---|---|---|
|지도|Deck.gl Scatterplot|JOIN Dataset|
|응급실 가용 포화여부|Table|er_realtime|
|공지 사항 병원 목록|Table|er_realtime|
|서울 구별 운영 병원|Bar|er_hospitals|
|서울 시간대별 포화율|Mixed Chart|er_hourly_stats|
|도시별 포화 vs 수용|Mixed Chart|er_realtime|
|지역별 평균 포화율|Pie|er_realtime|

## 발견한 인사이트

```
가장 위험한 시간대:
  18시 포화율 최고 / 평균 가용병상수 최저
  16~17시부터 꾸준히 상승 → 19시 이후 점차 감소

가장 위험한 지역 (포화율):
  서울  23.48%
  인천  23.96% (병원 수 적은데 포화율 높음)
  전북  14.97%
  대구  14.26%

가장 위험한 병원:
  삼성서울병원         hvec = -26
  분당서울대학교병원   hvec = -14

서울 구별 응급의료기관:
  영등포구 9개 1위 (duty_eryn='1' 필터 적용)
  강남구는 일반 병원 많지만 응급의료기관은 적음

지도 색상:
  초록  hvec > 10        응급실 여유 있음
  노랑  0 <= hvec < 10   응급실 포화 직전
  빨강  hvec <= 0        응급실 포화 상태
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|DB 연결 실패|localhost 로 연결|`postgres:5432` 로 변경|
|Dataset 테이블 안 보임|Schema 가 public 아님|Schema → public 선택|
|차트 데이터 없음|er_realtime 데이터 없음|Producer + Consumer 실행 확인|
|지도 차트 에러|wgs84_lat/lon NULL|`WHERE wgs84_lat IS NOT NULL` 추가|
|JOIN 차트 에러|er_hospitals 미적재|06_Hospital_Airflow 완료 후 시도|
|region 다 서울|서울 병원 위주 수집|정상 / 전국 수집 시 해소|
|mk 컬럼 에러|VARCHAR(1) 초과 값|VARCHAR(10) 변경 완료 확인|
|새로고침해도 그대로|자동 새로고침 미설정|대시보드 → Set auto-refresh 30초|

✅ 완료되면 → [[06_Hospital_Airflow]] 로 이동 (DAG 실행 후 지역별 차트 추가)