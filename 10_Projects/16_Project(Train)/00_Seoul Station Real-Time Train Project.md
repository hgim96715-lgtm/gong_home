>공공데이터 API 로 서울역 실시간 열차 정보를 수집해서, 실제 역 전광판처럼 보여주는 실시간 데이터 파이프라인 구축

## 시작하게 된 이유

기차역에 가면 전광판에 실시간으로 열차 정보가 뜨는것을 확인했을때 
지연됐는지, 몇 분 후에 들어오는지, 어느 플랫폼인지 실시간으로 반영 되는 게 너무 신기했다.
그러면서 자연스럽게 궁금증이 생겼다.  **데이터는 어디서 오는 걸까? 어떻게 실시간으로 업데이트가 되는 걸까?** 공부를 하다보니 이게 결국 데이터 파이프라인 이라는 것을 공부하면서 알게되었고  내가 직접 해보고 싶다는 생각이 들어 시작하게 되었다.

---
## 목표 

```text
공공데이터 API (KORAIL)
        ↓
  Python Producer    ← 주기적으로 API 호출
        ↓
      Kafka          ← 실시간 메시지 스트림
        ↓
 Spark Streaming     ← 데이터 정제 / 가공
        ↓
   PostgreSQL        ← 저장
        ↓
    Streamlit        ← 실시간 대시보드 (역 전광판처럼)
        ↓
   Airflow           ← 전체 파이프라인 스케줄링 / 모니터링
```

---
## 데이터 소스 

>[한국철도공사_열차운행정보 ](https://www.data.go.kr/data/15125762/openapi.do)

```text
공공데이터포털 (data.go.kr)
→ 한국철도공사_열차운행정보 조회서비스
→ 서울역 출발 / 도착 열차 실시간 정보
→ API 키 발급 후 무료 사용
```

### 제공 데이터 2종

#### ① 여객열차 운행계획

```text
운행일자, 열차번호, 출발역코드, 출발역명,
도착역코드, 도착역명, 열차계획출발일시, 열차계획도착일시
→ 오늘 어떤 열차가 어디서 어디로 갈 예정인지
```

#### ② 여객열차 운행정보

```text
운행일자, 열차번호, 열차운행일련번호, 역코드, 역명,
주운행선코드, 주운행선명, 상행하행구분코드,
정차구분코드, 정차구분명, 열차출발일시, 열차도착일시
→ 실제로 열차가 각 역을 몇 시에 출발/도착했는지 (실시간)
```

>① 운행계획 = 예정 시각  (스케줄)
>② 운행정보 = 실제 시각  (실시간)

---
## 기술 스택

| 역할     | 기술                |
| ------ | ----------------- |
| 데이터 수집 | Python (requests) |
| 메시지 큐  | Kafka (KRaft 모드)  |
| 스트림 처리 | Spark Streaming   |
| 저장소    | PostgreSQL        |
| 대시보드   | Superset (실시간 BI) |
| 스케줄링   | Airflow           |
| 컨테이너   | Docker Compose    |

---
## 단계별 진행 순서

- **STEP 1.** Docker Compose 기초 세팅 [[01_Docker_Setup_Postgresql_Setup]]
    - Kafka + PostgreSQL 컨테이너 구성 (Kafka + PostgreSQL 먼저 Docker Set up , 나중에 하나씩 추가)
- [ ]  **STEP 2.** 공공데이터 API 연동 [[02_API_Producer]]
    - Python 으로 서울역 열차 데이터 가져오기
- [ ]  **STEP 3.** Kafka Producer 구성
    - Python → Kafka topic 으로 데이터 발행
- [ ]  **STEP 4.** Spark Streaming Consumer
    - Kafka → Spark → 데이터 정제 → PostgreSQL 저장
- [ ]  **STEP 5.** Superset 대시보드
    - PostgreSQL 연동 → 실시간 전광판 UI
- [ ]  **STEP 6.** Airflow 연동
    - 파이프라인 스케줄링 + 모니터링
- [ ]  **STEP 7.** 전체 통합 테스트 + README 정리

---
## 프로젝트 폴더 구조

```text
seoul-train-realtime/
├── docker-compose.yml       ← 전체 컨테이너 정의
├── .env                     ← API 키, DB 설정 등
│
├── producer/                ← STEP 2, 3
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
│
├── spark/                   ← STEP 4
│   ├── Dockerfile
│   └── streaming_job.py
│
├── airflow/                 ← STEP 6
│   └── dags/
│       └── train_pipeline.py
│
├── superset/                ← STEP 5
│   └── superset_config.py
│
├── postgres/
│   └── init.sql
│
└── docs/
    ├── README.md
    └── architecture.png
```

---
## 목표 대시보드 화면

```text
┌─────────────────────────────────────────┐
│         서울역 실시간 열차 현황           │
│              21:15:30                   │
├──────┬──────────┬──────┬────────────────┤
│ 열차 │  행선지   │ 출발 │     상태       │
├──────┼──────────┼──────┼────────────────┤
│ KTX  │  부산    │21:20 │ 🟢 정상운행    │
│ KTX  │  광주    │21:35 │ 🔴 15분 지연   │
│ ITX  │  춘천    │21:40 │ 🟢 정상운행    │
└──────┴──────────┴──────┴────────────────┘
```

---

