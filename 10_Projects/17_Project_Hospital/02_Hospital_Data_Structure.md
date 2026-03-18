---
aliases:
  - Hospital 데이터 구조
  - API 응답 확인
  - er_realtime
  - er_hospitals
tags:
  - Project
related:
  - "[[00_Hospital_Project]]"
  - "[[Python_URL_Parsing]]"
  - "[[Python_ElementTree]]"
  - "[[Python_Requests_Methods]]"
  - "[[Python_Requests_Response]]"
  - "[[Python_OS_Module]]"
  - "[[PostgreSQL_Setup]]"
  - "[[CS_REST_API_Methods]]"
---
# 02_Hospital_Data_Structure — API 응답 확인 + 테이블 설계

## 이 단계의 목표

```
① API 실제 응답 확인 → 어떤 컬럼이 오는지 파악
② 컬럼 분석 → 우리가 필요한 것만 선별
③ init.sql 작성 → 테이블 생성
④ Docker 재시작 → 테이블 확인
```

```
공공데이터포털 미리보기:
  마이페이지 → 데이터활용 → Open API → 활용신청 현황 → 미리보기
```

---

---

# API 엔드포인트 정리

```
기본 URL: http://apis.data.go.kr/B552657/ErmctInfoInqireService/

실시간 수집 (Producer 5분마다):
  ① /getEmrrmRltmUsefulSckbdInfoInqire   응급실 실시간 가용병상
  ② /getSrsillDissAceptncPosblInfoInqire  중증질환자 수용가능정보
  ③ /getEmrrmSrsillDissMsgInqire          응급실 메시지

정적 수집 (Airflow 1일 1회):
  ④ /getEgytBassInfoInqire               응급의료기관 기본정보
```

---

---

# ① 응급실 실시간 가용병상 API

**엔드포인트:** `/getEmrrmRltmUsefulSckbdInfoInqire` **저장 테이블:** `er_realtime`

### 요청 파라미터

|항목명|영문|필수|샘플|설명|
|---|---|---|---|---|
|주소(시도)|STAGE1|필수|서울특별시|빈값으로 전국 조회 테스트 필요|
|주소(시군구)|STAGE2|필수|강남구|빈값 가능 여부 확인 필요|
|페이지 번호|pageNo|옵션|1||
|목록 건수|numOfRows|옵션|10||

```text
✅ 테스트 결과 확인:
  STAGE1 / STAGE2 없이 pageNo=1 로 호출
  → 강릉아산병원 등 전국 데이터 반환
  → 전국 조회 가능

  API 문서에 "필수" 표기는 잘못된 것
  → STAGE 없이 numOfRows 최대로 해서 전체 수집
  

✅ 전체 병원수 :417 확인 
```


### 테스트 코드

```python
import requests
import xml.etree.ElementTree as ET
from dotenv import load_dotenv
import os
from urllib.parse import unquote,quote

load_dotenv()

raw_key = os.getenv("HOSPITAL_API_KEY")
decoded_key = unquote(raw_api_key)   # 이중 인코딩 방지

BASE_URL = "http://apis.data.go.kr/B552657/ErmctInfoInqireService"

url = f"{BASE_URL}/getEmrrmRltmUsefulSckbdInfoInqire"
params = {
    "serviceKey": decoded_key,
    # STAGE1 / STAGE2 없어도 전국 조회 가능 
    "numOfRows":  "10",
    "pageNo":     "1"
}

try:
    r = requests.get(url, params=params, timeout=10)
    r.raise_for_status()
    root = ET.fromstring(r.content)
    
    # print(f" API는 뭘 보냈을까? {r.content.decode('utf-8')}")
    total = root.findtext(".//totalCount", "0")
    print(f"전체 병원 수: {total}")

    item = root.find(".//item")
    if item is not None:
        for child in item:
            print(f"{child.tag}: {child.text}")
    else:
        print("⚠️ 데이터 없음 → STAGE1 필수일 수 있음")

except RequestException as e:
    print(f"네트워크 에러: {e}")
except ET.ParseError:
    print("XML 파싱 에러")
```

### 응답 컬럼 목록

|항목명|영문|샘플|설명|
|---|---|---|---|
|기관코드|hpid|A0000028|병원 ID|
|입력일시|hvidate|2013-10-01|데이터 갱신 시각|
|**응급실**|**hvec**|**32**|**응급실 가용병상 ← 핵심**|
|수술실|hvoc|3||
|신경중환자|hvcc|0||
|신생중환자|hvncc|0||
|흉부중환자|hvccc|8||
|일반중환자|hvicc|8||
|입원실|hvgc|253||
|CT가용|hvctayn|Y|Y/N|
|MRI가용|hvmriayn|Y|Y/N|
|조영촬영기가용|hvangioayn|Y|Y/N|
|인공호흡기가용|hvventiayn|Y|Y/N|
|구급차가용|hvamyn|Y|Y/N|
|응급실 당직의 직통|hv1|02-3410-2062||
|내과중환자실|hv2|3||
|외과중환자실|hv3|6||
|VENTI(소아)|hv10|Y|Y/N|
|인큐베이터|hv11|Y|Y/N|
|기관명|dutyname|삼성서울병원||
|응급실전화|dutytel3|02-3410-2060||

---

---

# ② 중증질환자 수용가능정보 API

**엔드포인트:** `/getSrsillDissAceptncPosblInfoInqire` **저장 테이블:** `er_realtime` (① 과 hpid 병합)

### 요청 파라미터

|항목명|영문|필수|샘플|설명|
|---|---|---|---|---|
|주소(시도)|STAGE1|필수|서울특별시||
|주소(시군구)|STAGE2|필수|강남구||
|질환/수술명|SM_TYPE|옵션|1|mkioskty Y 인 병원 찾기|
|페이지 번호|pageNo|옵션|1||
|목록 건수|numOfRows|옵션|10||

### 응답 컬럼 목록

|항목명|영문|샘플|설명|
|---|---|---|---|
|기관명|dutyName|삼성서울병원||
|기관ID|hpid|A0000028|hpid 기준으로 ① 과 병합|
|**Gate keeper**|**mkioskty25**|**Y**|**중증외상 ← 핵심**|
|뇌출혈수술|mkioskty1|Y|Y:가능 / N:불가|
|뇌경색의재관류|mkioskty2|Y||
|심근경색의재관류|mkioskty3|Y||
|복부손상의수술|mkioskty4|Y||
|사지접합의수술|mkioskty5|N||
|응급내시경|mkioskty6|Y||
|응급투석|mkioskty7|Y||
|조산산모|mkioskty8|Y||
|정신질환자|mkioskty9|N||
|신생아|mkioskty10|Y||
|중증화상|mkioskty11|N||

---

---

# ③ 응급실 메시지 API

**엔드포인트:** `/getEmrrmSrsillDissMsgInqire` **저장 테이블:** `er_realtime.notice_msg`

```
병원이 직접 입력하는 공지 메시지
병상 수만으로는 파악 안 되는 실제 상황 포착

예시:
  "현재 응급실 포화로 경증 환자 수용 불가"
  "외상외과 전문의 부재로 외상 수술 불가"
  "야간 소아과 전문의 없음"
```

### 요청 파라미터

|항목명|영문|필수|샘플|설명|
|---|---|---|---|---|
|기관ID|hpid|필수|A0000028|병원 ID|
|기관명|hpNm|옵션|삼성서울병원||
|주소|hpAddr|옵션|서울||

### 응답 컬럼

| 항목명     | 영문        | 설명                     |
| ------- | --------- | ---------------------- |
| 기관ID    | hpid      | hpid 기준으로 ① 과 병합       |
| 기관명     | hpName    |                        |
| 응급실 메시지 | symBlkMsg | 공지 메시지 → notice_msg 저장 |

---

---

# ④ 응급의료기관 기본정보 API

**엔드포인트:** `/getEgytBassInfoInqire` **저장 테이블:** `er_hospitals`

```
Airflow 1일 1회 수집
MKioskTy = "이 병원이 할 수 있는 것" (인증된 역량, 정적)
  vs
실시간 mkioskty = "지금 받을 수 있나" (현재 상황, 동적)
```

### 요청 파라미터

|항목명|영문|필수|샘플|설명|
|---|---|---|---|---|
|기관ID|HPID|옵션|A0000028|없으면 전체 조회|
|페이지 번호|pageNo|옵션|1||
|목록 건수|numOfRows|옵션|10||

### 응답 컬럼 목록

|항목명|영문|샘플|설명|
|---|---|---|---|
|기관ID|hpid|A0000028|PK|
|기관명|dutyName|세브란스병원||
|주소|dutyAddr|서울특별시 강남구...|region 추출용|
|대표전화|dutyTel1|02-3410-2114||
|**응급실전화**|**dutyTel3**|**02-3410-2060**||
|응급실운영여부|dutyEryn|1|1:운영 / 2:미운영|
|병상수|dutyHano|1966||
|총병상수|hpbdn|2086||
|**병원위도**|**wgs84Lat**|**37.488**|지도용|
|**병원경도**|**wgs84Lon**|**127.085**|지도용|
|Gate keeper|MKioskTy25|Y|중증외상 (인증)|
|뇌출혈수술|MKioskTy1|Y|Y:가능 / N:불가|
|뇌경색의재관류|MKioskTy2|Y||
|심근경색의재관류|MKioskTy3|Y||
|복부손상의수술|MKioskTy4|Y||
|응급내시경|MKioskTy6|Y||
|응급투석|MKioskTy7|Y||
|신생아|MKioskTy10|Y||
|흉부중환자실|hpccuyn|15||
|신경중환자실|hpcuyn|10||
|응급실(수)|hperyn|47||
|수술실|hpopyn|38||

---

---

# ⑤ 우리가 쓸 컬럼 선별

## er_realtime (① + ② + ③ hpid 병합)

|DB 컬럼|API 필드|출처 API|설명|
|---|---|---|---|
|hpid|hpid|①|병원 ID|
|hpname|dutyname|①|병원명|
|**hvec**|**hvec**|**①**|**응급실 가용병상 ← 핵심**|
|hvoc|hvoc|①|수술실 가용 수|
|hvctayn|hvctayn|①|CT 가용 (Y/N)|
|hvventiayn|hvventiayn|①|인공호흡기 가용 (Y/N)|
|hv_stroke|mkioskty1|②|뇌출혈수술 수용 (실시간)|
|hv_cardiac|mkioskty3|②|심근경색 수용 (실시간)|
|hv_trauma|mkioskty25|②|중증외상 수용 (실시간)|
|hv_pediatric|mkioskty10|②|신생아 수용 (실시간)|
|duty_addr|dutyAddr|①|주소 (region 추출용)|
|region|-|-|시도명 (주소 앞 2자리 파싱)|
|sym_blk_msg|symBlkMsg|③|응급실 공지 메시지|

## er_hospitals (④)

|DB 컬럼|API 필드|설명|
|---|---|---|
|hpid|hpid|병원 ID (PK)|
|hpname|dutyName|기관명|
|duty_addr|dutyAddr|주소|
|duty_tel|dutyTel3|응급실 전화|
|duty_eryn|dutyEryn|응급실 운영여부|
|wgs84_lat|wgs84Lat|위도|
|wgs84_lon|wgs84Lon|경도|
|hpbdn|hpbdn|전체 병상 수|
|mk_stroke|MKioskTy1|뇌출혈수술 가능 (인증)|
|mk_cardiac|MKioskTy3|심근경색 가능 (인증)|
|mk_trauma|MKioskTy25|중증외상 Gate keeper (인증)|
|mk_pediatric|MKioskTy10|신생아 가능 (인증)|
|region|-|시도명 (dutyAddr 파싱)|

---

---

# ⑥ postgres/init.sql 작성

>Y/N은 VARCHAR로 할까 CHECK 으로 할까? 
>API 서버가 갑자기 에러가 나서 빈칸("")이나 미상("U") 같은 예기치 못한 값을 줄때 파이프라인이 뻗지 않게 하기위해서 VARCHAR(1)로 하기로 결정 
>[[PostgreSQL_Setup#② 자주 쓰는 자료형]] 참고 

>region은 혹시 모르니 저번에 train시 21개 로 막힌 오류가 있어서 50으로 정리 

```sql
-- ① er_realtime: 응급실 실시간 병상 현황 (5분마다 적재)
--    API: ① 가용병상 + ② 중증질환 + ③ 메시지 → hpid 병합 → 1 row
CREATE TABLE IF NOT EXISTS er_realtime (
    id           SERIAL PRIMARY KEY,
    hpid         VARCHAR(20),
    hpname       VARCHAR(100),
    hvec         INT,            -- 응급실 가용병상 수 ← 핵심 (API ①)
    hvoc         INT,            -- 수술실 가용 수 (API ①)
    hvctayn      VARCHAR(1),     -- CT 가용 Y/N (API ①)
    hvventiayn   VARCHAR(1),     -- 인공호흡기 가용 Y/N (API ①)
    hv_stroke    VARCHAR(1),     -- 뇌출혈수술 수용 Y/N (API ②)
    hv_cardiac   VARCHAR(1),     -- 심근경색 수용 Y/N (API ②)
    hv_trauma    VARCHAR(1),     -- 중증외상 수용 Y/N (API ②)
    hv_pediatric VARCHAR(1),     -- 신생아 수용 Y/N (API ②)
    duty_addr    VARCHAR(200),   -- 주소 (API ①)
    region       VARCHAR(50),    -- 시도명 (주소 앞 2자리 파싱)
    notice_msg  TEXT,           -- 응급실 공지 메시지 (API ③)
    data_type    VARCHAR(20),    -- 'er_realtime'
    created_at   TIMESTAMP DEFAULT NOW()
);

-- ② er_hourly_stats: 시간대별 집계 (Airflow 배치)
CREATE TABLE IF NOT EXISTS er_hourly_stats (
    id              SERIAL PRIMARY KEY,
    stat_hour       TIMESTAMP,
    region          VARCHAR(50),
    avg_beds        NUMERIC(5,2),   -- 평균 가용병상
    zero_count      INT,            -- 병상 0개 병원 수
    total_hospitals INT,
    saturation_pct  NUMERIC(5,2),   -- 포화율 (zero_count / total * 100)
    created_at      TIMESTAMP DEFAULT NOW()
);

-- ③ er_hospitals: 병원 기본정보 (Airflow 1일 1회)
--    API: ④ 응급의료기관 기본정보 조회 (getEgytBassInfoInqire)
--    MKioskTy = 인증된 역량 (정적) ← 실시간 hv_* 와 다름
--  여기서는 hpid로 primary key, 굳이 id가 필요없음 
CREATE TABLE IF NOT EXISTS er_hospitals (
    hpid         VARCHAR(20) PRIMARY KEY,
    hpname       VARCHAR(100),
    duty_addr    VARCHAR(200),
    duty_tel     VARCHAR(20),     -- 응급실 전화 (dutyTel3)
    duty_eryn    VARCHAR(1),      -- 응급실 운영여부 (1:운영)
    wgs84_lat    NUMERIC(10,7),   -- 위도
    wgs84_lon    NUMERIC(10,7),   -- 경도 
    hpbdn        INT,             -- 전체 병상 수
    mk_stroke    VARCHAR(1),      -- 뇌출혈수술 가능 (인증)
    mk_cardiac   VARCHAR(1),      -- 심근경색 가능 (인증)
    mk_trauma    VARCHAR(1),      -- 중증외상 Gate keeper (인증)
    mk_pediatric VARCHAR(1),      -- 신생아 가능 (인증)
    region       VARCHAR(50),
    updated_at   TIMESTAMP DEFAULT NOW()
);
```

---

---

# ⑦ Docker 재시작 후 테이블 확인

```bash
# init.sql 은 최초 실행 시에만 자동 실행
# 볼륨이 이미 있으면 미실행 → down -v 로 초기화 후 재시작
docker compose down -v
docker compose up -d

# 테이블 생성 확인
docker exec -it hospital-postgres psql -U hospital_user -d hospital_db 
```

```
hospital_db=# \dt
                 List of relations
 Schema |       Name       | Type  |     Owner
--------+------------------+-------+---------------
 public | er_hospitals     | table | hospital_user
 public | er_hourly_status | table | hospital_user
 public | er_realtime      | table | hospital_user
```

---

---

# DataGrip 연결

```
Host:     localhost
Port:     5434
Database: hospital_db
User:     hospital_user
Password: hospital_password
```

---

---

# 설계 고민 노트

```
STAGE1/STAGE2 필수 표기 문제:
  API 문서에 "필수" 로 적혀있지만
  실제 테스트 결과 없어도 전국 조회 가능 ✅
  pageNo=1 호출 시 강릉아산병원 등 전국 데이터 반환 확인
  → Producer 에서 STAGE 없이 numOfRows 최대로 전체 수집

hv_* (실시간) vs mk_* (인증) 구분:
  hv_stroke (er_realtime) = 지금 뇌출혈 환자 받을 수 있나 (동적)
  mk_stroke (er_hospitals) = 이 병원이 뇌출혈 수술 인증받은 곳인가 (정적)

region 파싱:
  "서울특별시 강남구..." → 앞 2자리 → "서울"
  Producer 에서 Python 으로 처리

API ① + ② + ③ 병합:
  같은 hpid 끼리 dict 로 모아서 Kafka 에 1 row 발행
  Producer 에서 처리

Y/N 컬럼 VARCHAR(1) vs CHECK 결정:
  CHECK (col IN ('Y','N')) 으로 하면 제약이 강함
  BUT API 서버 에러 시 빈값("") 또는 미상값("U") 이 올 수 있음
  → 파이프라인이 뻗지 않게 VARCHAR(1) 로 결정
  → 데이터 검증은 Superset / 집계 쿼리에서 처리

er_realtime.hpid UNIQUE 여부:
  er_realtime 은 5분마다 같은 병원 데이터를 계속 쌓는 테이블
  → 같은 hpid 가 반복 등장 → UNIQUE 붙이면 안 됨 ❌
  → created_at 으로 시점 구분
  er_hospitals 은 hpid 가 PRIMARY KEY → 자동으로 UNIQUE ✅
```

----

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`hospital-postgres Exited (3)`|init.sql 문법 오류|`docker logs hospital-postgres` 로 에러 확인|
|`syntax error at or near ...`|SQL 문법 오류 (`;` 누락 등)|init.sql 세미콜론 확인|
|`column ... does not exist`|컬럼명 오타|init.sql 컬럼명 재확인|
|`type ... does not exist`|ENUM 타입 선언 누락 또는 순서 오류|`CREATE TYPE` 을 `CREATE TABLE` 위에 배치|
|테이블 안 생김|볼륨 이미 존재|`docker compose down -v` 후 재실행|



✅ 완료되면 → [[03_Hospital_Producer]] 으로 이동