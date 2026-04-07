
```
목표: 데이터 엔지니어 실무에서 dbt 로 데이터 변환 파이프라인 구축
     ELT 패턴 이해 + SQL 기반 모델링 + 테스트 자동화
```

---

---

## Level 0. dbt 란 무엇인가

```
dbt 를 쓰기 전에 이해해야 하는 핵심 개념
ETL vs ELT / dbt 가 왜 나왔는지
```

| 노트                    | 핵심 개념                                        |
| --------------------- | -------------------------------------------- |
| [[DBT_Overview]]      | ETL vs ELT / dbt 의 역할 / Transformation 계층    |
| [[DBT_Install_Setup]] | 설치 / profiles.yml / 프로젝트 초기화 / PostgreSQL 연결 |


---

---

## Level 1. 핵심 개념

```
dbt 프로젝트의 구조와 동작 원리 이해
```

|노트|핵심 개념|
|---|---|
|[[DBT_Project_Structure]]|dbt_project.yml / models / seeds / tests / macros 폴더 구조|
|[[DBT_Models]]|SELECT 만 쓰는 변환 모델 / .sql 파일 / ref() / source()|
|[[DBT_Materializations]]|table / view / incremental / ephemeral 차이|
|[[DBT_Sources]]|sources.yml / source() 함수 / freshness 체크|

---

---

## Level 2. 데이터 품질

```
dbt 의 핵심 강점 — 테스트 자동화
```

|노트|핵심 개념|
|---|---|
|[[DBT_Tests]]|not_null / unique / accepted_values / relationships|
|[[DBT_Schema_yml]]|schema.yml 작성법 / 컬럼 설명 / 테스트 설정|
|[[DBT_Custom_Tests]]|커스텀 테스트 / singular test / generic test|

---

---

## Level 3. 문서화 & 계보

```
데이터 계보(Lineage) 자동 생성
어디서 왔고 어디로 가는지 시각화
```

|노트|핵심 개념|
|---|---|
|[[DBT_Documentation]]|dbt docs generate / dbt docs serve / 설명 작성|
|[[DBT_Lineage]]|DAG 시각화 / ref() 의존성 / 모델 계보|

---

---

## Level 4. 고급 기능

```
실무에서 자주 쓰는 고급 패턴
```

|노트|핵심 개념|
|---|---|
|[[DBT_Macros]]|Jinja 문법 / macro 정의 / 재사용 가능한 SQL|
|[[DBT_Incremental]]|incremental 전략 / is_incremental() / unique_key|
|[[DBT_Seeds]]|CSV → DB 테이블 / dbt seed / 코드 관리 매핑 테이블|
|[[DBT_Snapshots]]|SCD Type 2 / 이력 관리 / dbt snapshot|
|[[DBT_Variables]]|var() / --vars / 환경별 분기 처리|

---

---

## Level 5. 실무 패턴

```
데이터 엔지니어가 실무에서 쓰는 dbt 패턴
```

|노트|핵심 개념|
|---|---|
|[[DBT_Layer_Design]]|staging / intermediate / mart 레이어 설계|
|[[DBT_Airflow]]|Airflow + dbt 연동 / BashOperator / dbt-airflow|
|[[DBT_PostgreSQL]]|PostgreSQL 에서 dbt 사용 / 실습 환경 구성|

---

---

## 자주 쓰는 명령어

```bash
dbt init 프로젝트명          # 프로젝트 생성
dbt debug                   # 연결 테스트
dbt run                     # 모든 모델 실행
dbt run --select 모델명      # 특정 모델만
dbt test                    # 테스트 실행
dbt seed                    # CSV → DB 적재
dbt docs generate           # 문서 생성
dbt docs serve              # 문서 서버 실행 (localhost:8080)
dbt snapshot                # 스냅샷 실행
dbt compile                 # SQL 컴파일 (실행 안 함)
```