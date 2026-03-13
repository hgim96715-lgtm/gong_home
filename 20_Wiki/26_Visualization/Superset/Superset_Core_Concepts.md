---
aliases:
  - Superset 구조
  - Superset 핵심 개념
  - Database Dataset Chart Dashboard
  - BI툴
tags:
  - Superset
related:
  - "[[00_Superset_HomePage]]"
  - "[[Superset_Database_Connection]]"
  - "[[Superset_Dataset]]"
---

# Superset_Core_Concepts — 핵심 구조 이해

## 한 줄 요약

```
Superset = DB 에 연결 → 데이터 등록 → 차트 생성 → 대시보드 조합
```

---

---

# ① Superset 이란

```
Apache Superset — 오픈소스 데이터 시각화 / BI(Business Intelligence) 툴
Airbnb 에서 만들어 2016년 오픈소스로 공개
SQL 을 알면 바로 쓸 수 있음

주요 특징:
  다양한 DB 지원  PostgreSQL / MySQL / BigQuery / Redshift / Snowflake 등
  SQL Lab       SQL 직접 실행 + 결과 탐색
  차트 30+종     Bar / Pie / Line / Table / Map 등
  대시보드       차트 조합 + 자동 새로고침 + 필터 인터랙션
  접근 권한 관리  역할(Role) 기반 사용자 권한 설정
```

---

---

# ② 핵심 구성 요소 — 4계층

```
Database
  └── Dataset
        └── Chart
              └── Dashboard
```

```
DB 연결 없이는 아무것도 못 함
Dataset 없이는 Chart 못 만듦
Chart 없이는 Dashboard 못 만듦
→ 반드시 위에서 아래로 순서대로 만들어야 함
```

---

---

# ③ Database — DB 연결

```
Superset 이 외부 DB 에 접속하기 위한 연결 정보

SQLAlchemy URI 형식으로 연결:
  postgresql://user:password@host:port/dbname
  mysql://user:password@host:port/dbname

하나의 Superset 에 여러 DB 연결 가능
Settings → Database Connections 에서 관리
```

```
Database 는 "연결 설정" 만 저장
실제 데이터를 Superset 안으로 복사하는 게 아님
쿼리할 때마다 연결해서 가져오는 방식
```

---

---

# ④ Dataset — 차트의 재료

```
Chart 를 만들 때 사용할 데이터 원본을 등록하는 것
Database 에서 특정 테이블 또는 SQL 을 Dataset 으로 등록

두 가지 종류:
  물리 Dataset   DB 의 실제 테이블을 그대로 등록
  가상 Dataset   SQL 을 작성해서 뷰처럼 등록
```

```
물리 Dataset:
  train_realtime 테이블 → Dataset 으로 등록
  → 모든 컬럼이 차트에서 바로 사용 가능

가상 Dataset:
  SELECT mrnt_nm, COUNT(*) AS cnt FROM train_realtime GROUP BY mrnt_nm
  → 이 SQL 결과를 Dataset 으로 등록
  → 복잡한 집계를 미리 Dataset 에 담아두는 방식
```

```
Dataset 에서 할 수 있는 것:
  컬럼 타입 지정    날짜인지 숫자인지 문자인지
  Metric 정의      COUNT(*), AVG(delay), SUM(cnt) 등 자주 쓰는 집계 저장
  필터 기본값 설정  대시보드 필터와 연동
```

---

---

# ⑤ Chart — 시각화 단위

```
Dataset 을 기반으로 만드는 시각화 하나
Bar Chart, Pie Chart, Line Chart, Table 등 30+ 종류

Chart 를 만들 때 설정:
  Chart type  어떤 종류의 차트인지
  Dataset     어떤 데이터를 쓸 것인지
  Dimension   X축 / 분류 기준 (mrnt_nm, status 등)
  Metric      Y축 / 집계값 (COUNT(*), AVG(arr_delay) 등)
  Filter      조건 (오늘 날짜만, 특정 노선만 등)
```

```
Chart 는 Dataset 에 종속됨
Dataset 을 바꾸면 Chart 도 영향을 받음
여러 Dashboard 에서 같은 Chart 를 재사용 가능
```

---

---

# ⑥ Dashboard — 차트 모음

```
여러 Chart 를 하나의 화면에 배치한 것
드래그 앤 드롭으로 레이아웃 구성

Dashboard 기능:
  자동 새로고침   10초 / 30초 / 1분 등 설정 가능
  필터 연동       Native Filter 로 여러 Chart 동시 필터링
  공유            URL 공유 / 이미지 다운로드 / PDF 내보내기
  크로스 필터     차트 클릭 시 다른 차트도 연동 필터링
```

---

---

# ⑦ SQL Lab — 자유 탐색 도구

```
Dashboard / Chart 와 별개로
SQL 을 직접 작성해서 DB 를 탐색하는 공간

SQL Lab 에서 할 수 있는 것:
  쿼리 직접 실행    결과를 테이블로 바로 확인
  쿼리 저장         자주 쓰는 쿼리 북마크
  차트로 내보내기   SQL 결과를 바로 Chart 로 저장
  가상 Dataset 생성 SQL 을 Dataset 으로 등록
```

```
데이터 탐색 → SQL Lab
반복 사용    → Dataset + Chart 로 고정
```

---

---

# ⑧ 전체 흐름 요약

```
1. Database 등록
   Settings → Database Connections
   SQLAlchemy URI 입력 → Test Connection → 저장

2. Dataset 등록
   Data → Datasets → + Dataset
   Database 선택 → Schema 선택 → Table 선택

3. Chart 생성
   Charts → + Chart
   Dataset 선택 → Chart type 선택 → Dimension / Metric 설정

4. Dashboard 구성
   Dashboards → + Dashboard
   Chart 드래그 앤 드롭 → 자동 새로고침 설정

(탐색/검증은 언제든지 SQL Lab)
```