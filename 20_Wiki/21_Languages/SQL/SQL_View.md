---
aliases:
  - 가상테이블
  - Materialized_View
  - SQL view
  - 보안성
  - 독립성
  - 편리성
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_SubQuery]]"
  - "[[SQL_WITH_CTE]]"
---
# SQL View

## 개념 한 줄 요약 (Concept Summary)

실제 데이터는 하나도 가지고 있지 않으면서, 복잡한 `SELECT` 쿼리 문장을 저장해 두고 마치 **'진짜 테이블'** 인 것처럼 쓸 수 있게 해주는 **"가상 테이블(Virtual Table)"** 입니다.

---

## ⭐️ 뷰(VIEW)의 3대 핵심 특징

**보안성 (Security):**

- 원본 테이블에 있는 주민번호, 연봉 같은 민감한 컬럼을 빼고, 보여주고 싶은 컬럼만 골라서 뷰를 만들 수 있습니다.
- 사용자에게 원본 테이블 대신 뷰에 대한 접근 권한만 주면 완벽한 보안 통제가 가능합니다.

**독립성 (Logical Data Independence):**

- 원본 테이블의 구조가 바뀌더라도(예: 테이블이 쪼개지거나 컬럼이 추가됨), 뷰를 수정해서 기존과 똑같은 형태의 출력을 유지할 수 있습니다.
- 즉, 뷰를 바라보고 있는 애플리케이션이나 대시보드는 원본 DB 구조 변화에 영향을 받지 않고 **독립적**으로 돌아갑니다.

**편리성 (Convenience):**

- `JOIN`이 5개씩 걸리고 `GROUP BY`가 들어간 수십 줄짜리 복잡한 쿼리를 뷰로 예쁘게 포장해 두면, 다음부터는 `SELECT * FROM 뷰이름` 단 한 줄로 똑같은 결과를 뽑아낼 수 있습니다.

---

## 왜 필요한가 (Why is it needed)

**문제점:** 매번 조인(JOIN)이 잔뜩 걸린 복잡한 쿼리를 칠 때마다 오타가 나고, 외부인에게 DB를 열어줄 때 데이터 유출 사고의 위험이 큽니다. **해결책:** 복잡한 로직과 필터링이 완료된 안전한 형태의 쿼리를 `VIEW`라는 이름표를 붙여 DB에 영구 저장해 두고, 필요할 때마다 꺼내 씁니다.

---

## 실전 맥락 (Practical Context)

데이터 파이프라인을 구축하거나 Streamlit, Superset 같은 시각화 대시보드를 연결할 때, DB의 원본 테이블을 직접 찌르는 경우는 거의 없습니다.
대신 분석하기 좋게 잘 정제된(JOIN, 필터링 완료) 형태의 `VIEW`를 만들어 두고, 대시보드는 이 가상 테이블만 쳐다보게 만드는 것이 현업 데이터 엔지니어들의 **국룰 아키텍처**입니다.

---

---

# ① 기본 문법 — CREATE VIEW

```sql
CREATE VIEW 뷰이름 AS
SELECT 쿼리...;
```

```
동작 원리:
  AS 뒤에 적힌 SELECT 쿼리 텍스트(로직) 자체를 DB 딕셔너리에 저장
  데이터(행)를 복사해서 저장하는 것이 아님

  → 뷰를 조회할 때마다 저장된 SELECT 를 그때그때 실행
  → 원본 테이블 데이터가 바뀌면 뷰 결과도 실시간으로 반영 (거울)
```

```sql
-- 예시: 민감한 연봉·주민번호는 빼고 조회용 뷰 만들기
CREATE VIEW v_emp_public_info AS
SELECT
    e.사원번호,
    e.직원명,
    d.부서명
FROM 직원 e
JOIN 부서 d ON e.부서코드 = d.부서코드;
-- 이 순간 데이터가 복사되는 게 아니라 위 쿼리 텍스트만 저장됨

-- 사용 — 마치 진짜 테이블처럼
SELECT * FROM v_emp_public_info
WHERE 부서명 = '데이터엔지니어링팀';
```

---

## ⚠️ 여기서 AS 는 별칭(Alias)이 아니다

```sql
-- 테이블 별칭 AS (Alias)
SELECT e.사원번호 FROM 직원 AS e;
--                        ↑
--                  테이블 이름을 'e' 로 줄여부름 (임시 이름)

-- CREATE VIEW 의 AS
CREATE VIEW v_active_exhibitions AS
SELECT * FROM raw_exhibitions WHERE ...;
--                             ↑
--             "이 뷰의 정체는 이 SELECT 쿼리다" 라는 정의 연결자
--             별칭이 아님. "~로 정의한다" 는 의미
```

|상황|AS 의 역할|예시|
|---|---|---|
|`SELECT` 절|컬럼에 별칭 부여|`SELECT name AS 이름`|
|`FROM` 절|테이블에 별칭 부여|`FROM 직원 AS e`|
|`CREATE VIEW`|뷰의 내용을 정의하는 연결자|`CREATE VIEW v AS SELECT ...`|
|`CREATE TABLE AS`|CTAS — 테이블을 쿼리로 정의|`CREATE TABLE t AS SELECT ...`|

```
공통점: AS 뒤에 뭔가를 이어 붙인다
차이점:
  별칭 AS → 뒤에 이름이 옴
  정의 AS → 뒤에 SELECT 쿼리가 옴
```

---

# ② CREATE OR REPLACE VIEW

```sql
CREATE OR REPLACE VIEW v_active_exhibitions AS
SELECT *
FROM raw_exhibitions
WHERE is_active  = TRUE
  AND start_date <= CURRENT_DATE
  AND end_date   >= CURRENT_DATE;
```

```
CREATE VIEW           뷰가 이미 존재하면 → 에러
CREATE OR REPLACE VIEW 뷰가 이미 존재하면 → 덮어씌움 (에러 없음)
                       뷰가 없으면        → 새로 생성
```

```sql
-- ❌ 이미 존재하는 뷰에 CREATE VIEW 하면
CREATE VIEW v_active_exhibitions AS SELECT ...;
-- ERROR: relation "v_active_exhibitions" already exists

-- ✅ OR REPLACE 붙이면 안전하게 갱신
CREATE OR REPLACE VIEW v_active_exhibitions AS SELECT ...;
-- 기존 뷰 권한·의존성 유지하면서 내용만 교체됨
```

```
실무 원칙:
  처음 만들 때도 OR REPLACE 를 기본으로 쓰는 것이 편함
  → 나중에 수정할 때 DROP 없이 바로 덮어씌울 수 있음
  → init.sql 에서도 OR REPLACE 권장 (재실행 시 에러 방지)
```

### OR REPLACE 주의사항

```
컬럼 구조가 달라지면 제한이 있음

가능:
  WHERE 조건 변경
  컬럼 추가
  표현식 변경

불가 (DROP 후 재생성 필요):
  컬럼 순서 변경
  컬럼 타입 변경
  컬럼 삭제
```

---

# ③ COMMENT ON VIEW

```
뷰에도 COMMENT ON 으로 설명을 붙일 수 있음
테이블 COMMENT 와 동일한 문법
```

```sql
-- 뷰에 주석 달기
COMMENT ON VIEW v_active_exhibitions IS '현재 진행 중인 전시 정보 조회용 뷰';

-- 확인
SELECT obj_description('v_active_exhibitions'::regclass);

-- psql 에서 확인
\dv+   -- 뷰 목록 + COMMENT 내용
```

```
COMMENT ON 이 가능한 객체:
  TABLE    → COMMENT ON TABLE  테이블명 IS '설명'
  COLUMN   → COMMENT ON COLUMN 테이블명.컬럼명 IS '설명'
  VIEW     → COMMENT ON VIEW   뷰이름 IS '설명'
  INDEX    → COMMENT ON INDEX  인덱스명 IS '설명'
  DATABASE → COMMENT ON DATABASE DB명 IS '설명'
```

> COMMENT ON 상세 → [[SQL_DDL_Create#⑥ COMMENT ON — 메타데이터 주석 달기]]

---

# ④ 뷰 수정 / 삭제

```sql
-- 내용 수정 (OR REPLACE 로 덮어씌우기)
CREATE OR REPLACE VIEW v_emp_public_info AS
SELECT 사원번호, 직원명, 부서명, 입사일  -- 입사일 컬럼 추가
FROM ...;

-- 삭제 (원본 테이블 데이터는 절대 안 지워짐, 쿼리 텍스트만 삭제)
DROP VIEW v_emp_public_info;

-- 있으면 삭제, 없으면 무시
DROP VIEW IF EXISTS v_emp_public_info;

-- 이 뷰를 참조하는 다른 뷰까지 강제 삭제
DROP VIEW v_emp_public_info CASCADE;
```

---

# ⑤ 가상 테이블 3대장 비교

목적은 같지만 **"얼마나 오래 보존되는가(생명 주기)"** 에 따라 구분됩니다.

|구분|뷰 (VIEW)|WITH 구문 (CTE)|인라인 뷰 (Inline View)|
|---|---|---|---|
|정의|DB에 영구 저장된 쿼리 객체|쿼리 맨 위에서 임시 선언|FROM 절 괄호 안에서 즉석 생성|
|생성 키워드|`CREATE VIEW 뷰이름 AS`|`WITH 임시테이블명 AS`|`FROM (SELECT ...) AS 별칭`|
|수명|**DROP 하기 전까지 영구**|해당 SELECT 문 끝나면 소멸|해당 SELECT 문 끝나면 소멸|
|재사용성|누구나 언제든 재사용 가능|그 쿼리 스크립트 안에서만|재사용 불가 (1회용)|
|비유|법원 호적에 등록된 이름|회의 때만 쓸 별명 (포스트잇)|지나가던 행인 1 (이름 없음)|

```sql
-- 셋 다 같은 결과를 낼 수 있음

-- 1. VIEW (영구 저장)
CREATE VIEW v_seoul AS
SELECT * FROM raw_exhibitions WHERE location = '서울';
SELECT * FROM v_seoul;

-- 2. CTE (이 쿼리 안에서만)
WITH seoul AS (
    SELECT * FROM raw_exhibitions WHERE location = '서울'
)
SELECT * FROM seoul;

-- 3. 인라인 뷰 (딱 이 자리 1회)
SELECT * FROM (
    SELECT * FROM raw_exhibitions WHERE location = '서울'
) AS seoul;
```

---

# ⑥ 뷰 동작 원리 상세

뷰를 호출(`SELECT * FROM 뷰`)하면 DB 내부에서 일어나는 일:

```
1. 사용자가 SELECT * FROM v_active_exhibitions WHERE location = '서울'

2. DB가 딕셔너리에서 뷰의 원본 SQL 텍스트를 꺼냄
   → SELECT * FROM raw_exhibitions
     WHERE is_active = TRUE AND start_date <= CURRENT_DATE ...

3. 사용자 조건을 원본 SQL 과 합침 (Query Merge)
   → SELECT * FROM raw_exhibitions
     WHERE is_active = TRUE AND start_date <= CURRENT_DATE ...
       AND location = '서울'   ← 사용자 조건 추가됨

4. 합쳐진 쿼리로 원본 테이블 실제 조회
```

```
결론:
  원본 테이블 데이터가 바뀌면 뷰 결과도 실시간 반영
  뷰 자체는 데이터를 갖고 있지 않음
  조회할 때마다 SELECT 가 새로 실행됨 → 무거운 쿼리면 느릴 수 있음
  → 이 문제를 해결하는 것이 Materialized View
```

---

# ⑦ Materialized View (구체화된 뷰)

```
일반 VIEW    → 조회할 때마다 SELECT 를 새로 실행 (실시간, 느릴 수 있음)
Materialized → SELECT 결과를 디스크에 스냅샷으로 저장 (빠름, 실시간 아님)
```

```sql
-- 생성
CREATE MATERIALIZED VIEW mv_location_summary AS
SELECT
    location,
    COUNT(*) AS total_count,
    AVG(week_rank) AS avg_rank
FROM raw_exhibitions
WHERE is_active = TRUE
GROUP BY location;

-- 데이터 갱신 (수동으로 새로고침 필요)
REFRESH MATERIALIZED VIEW mv_location_summary;

-- 조회 중에도 갱신 허용 (락 없이 갱신)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_location_summary;

-- 삭제
DROP MATERIALIZED VIEW mv_location_summary;
```

|항목|VIEW|MATERIALIZED VIEW|
|---|---|---|
|데이터 저장|❌ 쿼리만 저장|✅ 결과를 디스크에 저장|
|실시간 반영|✅ 항상 최신|❌ REFRESH 해야 갱신|
|조회 속도|느림 (매번 실행)|빠름 (저장된 결과 읽기)|
|인덱스 생성|❌ 불가|✅ 가능|
|사용 패턴|실시간 조회|배치로 갱신, 빠른 대시보드|

```
실무 패턴:
  새벽 배치로 REFRESH MATERIALIZED VIEW 실행
  → 낮에 Superset 대시보드는 스냅샷 읽기만 → 빠름

  dbt 에서는 config(materialized='table') 로 동일 효과를 냄
```

---

# ⑧ 이 프로젝트에서 뷰를 최소화하는 이유

```
raw 레이어에서 뷰를 많이 만들면:
  dbt 와 관리 포인트가 겹침
  뷰 수정 vs dbt 모델 수정 어디서 해야 할지 혼란

이 프로젝트 전략:
  raw 레이어     → 뷰 최소화 (v_active_exhibitions 1개만)
  dbt staging   → 정제 로직
  dbt mart      → 분석용 최종 뷰 역할
  Superset      → dbt mart 만 바라봄
```

```sql
-- raw 레이어에 딱 1개만 유지하는 이유:
-- 크롤러 확인 / psql 에서 빠른 점검용으로만 씀
CREATE OR REPLACE VIEW v_active_exhibitions AS
SELECT *
FROM raw_exhibitions
WHERE is_active  = TRUE
  AND start_date <= CURRENT_DATE
  AND end_date   >= CURRENT_DATE;

COMMENT ON VIEW v_active_exhibitions IS '현재 진행 중인 전시 (raw 레이어 빠른 확인용)';
```

---

# 초보자가 자주 하는 실수

**"뷰를 통해서 원본 데이터를 수정(`INSERT`, `UPDATE`)할 수 있나요?"**

문법적으로 가능은 합니다. 하지만 실무에서는 거의 안 합니다. 뷰에 `GROUP BY`, `JOIN`, `DISTINCT` 등이 섞여 있으면 어떤 원본 테이블의 어느 행을 고쳐야 할지 DB가 판단할 수 없어서 에러가 납니다. 뷰는 **읽기 전용(Read-Only)** 으로 생각하는 것이 원칙입니다.

**"`CREATE VIEW` 와 `CREATE OR REPLACE VIEW` 를 언제 구분해서 써요?"**

실무에서는 처음부터 `OR REPLACE` 를 기본으로 쓰는 것이 편합니다. 나중에 수정할 때 DROP 없이 바로 덮어씌울 수 있고, init.sql 재실행 시 에러도 방지됩니다.

**"`AS` 가 별칭인가요?"**

`CREATE VIEW v AS SELECT ...` 에서 `AS` 는 별칭이 아니라 "이 뷰의 정체는 이 SELECT 쿼리다" 라는 **정의 연결자**입니다. `SELECT name AS 이름` 의 별칭 AS 와는 다릅니다.