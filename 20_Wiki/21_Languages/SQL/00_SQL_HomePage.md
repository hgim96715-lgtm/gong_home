### Level 0. 핵심 사고력 (Data Engineer's Mindset)

> **"쿼리를 짜기 전에 뇌를 먼저 SQL 모드로 세팅합니다."**

- [[01_SQL_Thinking_Roadmap]] : 🚨 쿼리 짜기 전 필독! (실행 순서 & 전략)
- [[SQL_Execution_Order]] : 작성 순서 vs 실행 순서 완벽 정리 (치트시트)
- [[SQL_Main_Table_Strategy]] : 쿼리의 주인공(FROM)을 정하는 절대 원칙

### Level 1. [SQLD 과목 1] 데이터 모델링의 이해

> **"데이터베이스의 기초 개념과 집을 짓는 설계도(ERD)를 배웁니다."**

- **1장. 데이터 모델링의 이해**
    - [[Concept_Database]] : DB란 무엇인가? (`DBMS`, `스키마`, `인스턴스`)
    - [[Concept_RDBMS]] : 관계형 데이터베이스(RDBMS)의 정체 (`테이블`, `행`, `열`, `무결성`)
    - [[Data_Modeling_Overview]] : 모델링의 3단계와 관점 (`개념 모델링`, `논리 모델링`, `물리 모델링`)
    - [[SQL_ERD_Components]] : 엔터티(Entity), 속성(Attribute), 관계(Relationship) (`카디널리티`, `식별/비식별`)
    - [[SQL_Keys_and_Identifiers]] : PK와 FK의 관계 (`본질식별자`, `인조식별자`, `주식별자`, `복합식별자`)

- **2장. 데이터 모델과 성능**
    - [[SQL_Normalization_정규화]] : 테이블을 쪼개는 기술, 정규화 (`이상현상`, `1NF`, `2NF`, `3NF`, `성능 데이터 모델링`)
    - [[SQL_De_Normalization_반정규화]] : 성능을 위해 다시 합치는 기술, 반정규화 (`테이블 병합/분할/추가`, `컬럼 반정규화`, `관계 반정규화`, `파티셔닝`)

---

### Level 2. [SQLD 과목 2 - 제1장] SQL 기본 (Basic)

> **"데이터를 조회하고, 가공하고, 테이블끼리 연결하는 필수 기술입니다."**

- **기초 조회 및 필터링**
    - [[Concept_What_is_SQL]] : SQL의 본질 (`집합적 사고방식`, `선언형 언어`, `DDL/DML/DCL/TCL`)
    - [[Concept_Table_Anatomy]] : 테이블 구조 해부 완벽 정리 (`행(Row)`, `열(Column)`, `도메인`)
    - [[SQL_SELECT_FROM]] : 데이터를 가져오는 가장 기본적인 구조 (`SELECT`, `FROM`, `별칭(AS)`, `*`)
    - [[SQL_Data_Types]] : 문자, 숫자, 날짜 타입의 이해 (`VARCHAR`, `NUMBER`, `DATE`, `BOOLEAN`, `NULL`)
    - [[SQL_Filtering_WHERE]] : 원하는 데이터만 쏙 골라내기 (`BETWEEN`, `IN`, `LIKE`, `IS NULL`, `AND/OR`)
    - [[SQL_ORDER_BY]] : 데이터 줄 세우기 (`ASC`, `DESC`, `NULL 정렬`, `다중 정렬`)

- **함수와 NULL의 이해**
    - [[SQL_Understanding_NULL]] : SQL의 영원한 골칫거리, NULL의 모든 것 (`3값 논리`, `NULL 연산`, `IS NULL`)
    - [[SQL_Numeric_Functions]] : 숫자형 함수 완벽 정리 (`ROUND`, `TRUNC`, `MOD`, `CEIL`, `FLOOR`)
    - [[SQL_String_Functions]] : 문자열 자르고 붙이기 (`LTRIM`, `RTRIM`, `REPLACE`, `SPLIT`, `TRIM`, `SUBSTR`)
    - [[SQL_Date_Functions]] : 날짜 다루기 (`NOW`, `CURRENT_DATE`, `INTERVAL`, `DATE_TRUNC`, `EXTRACT`)
    - [[SQL_Type_Casting]] : 명시적/암시적 형변환(`CAST`, `TO_CHAR`, `TO_DATE`)
    - [[SQL_NULL_Functions]] : 텅 빈 데이터(NULL) 심폐소생술 (`COALESCE`, `NVL`, `NULLIF`, `NVL2`)

- **그룹화 및 집계**
    - [[SQL_Aggregate_GROUP_BY]] : 그룹별 통계 내기 (`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`, `GROUP BY`)
    - [[SQL_DISTINCT_vs_GROUP_BY]] : 중복 제거 vs 묶어서 계산 (`DISTINCT`, `GROUP BY`)
    - [[HAVING_vs_WHERE]] : 필터링 시점의 차이 (`WHERE → 집계 전`, `HAVING → 집계 후`)
    - [[SQL_CASE_WHEN]] : IF-ELSE 로직 구현하기 (`CASE WHEN`, `THEN`, `ELSE`, `END`)

- **조인 (JOIN)**
    - [[SQL_JOIN_Concept]] : 데이터 연결의 본질과 원리 (`EQUI JOIN`, `NON EQUI JOIN`, `3개 이상 다중 조인`)
    - [[SQL_Standard_JOIN]] : ANSI 표준 조인 문법 (`INNER JOIN`, `LEFT/RIGHT/FULL OUTER JOIN`, `CROSS JOIN`, `NATURAL JOIN`)

---

### Level 3. [SQLD 과목 2 - 제2장] SQL 활용 (Advanced)

> **"복잡한 로직을 구현하고, 데이터를 입체적으로 분석하는 고급 기술입니다."**

- **서브쿼리 및 집합 연산**
    - [[SQL_SubQuery]] : 복잡한 쿼리 관리 (`인라인 뷰`, `스칼라`, `중첩 서브쿼리`, `EXISTS`, `IN`)
    - [[SQL_View]] : 쿼리를 저장하는 마법의 거울 (`가상 테이블`, `보안성`, `독립성`, `편리성`)
    - [[SQL_WITH_CTE]] : 쿼리를 변수처럼 조립하는 1회용 임시 테이블 (`WITH`, `CTE`, `재귀 CTE`, `가독성`)
    - [[SQL_UNION]] : 집합 연산자 (`UNION`, `UNION ALL`, `INTERSECT`, `MINUS/EXCEPT`)

- **분석 및 고급 집계**
    - [[SQL_Group_Functions]] : 소계와 총계 내기 (`ROLLUP`, `CUBE`, `GROUPING SETS`, `GROUPING`)
    - [[SQL_Window_Functions]] : 순위, 누적합, 이동평균의 마법 (`RANK`, `DENSE_RANK`, `ROW_NUMBER`, `PARTITION BY`, `OVER`, `LAG`, `LEAD`)
    - [[SQL_Top_N_Query]] : 상위 N개 데이터 추출하기 (`LIMIT`, `ROWNUM`, `ROW_NUMBER`, `RANK`, `DENSE_RANK`)
    - [[SQL_Self_Join]] : 하나의 테이블을 두 개인 것처럼 엮기 (`Self Join`, `사원-매니저`, `계층 비교`)
    - [[SQL_Hierarchical_Query]] : 꼬리에 꼬리를 무는 트리 구조 탐색 (`START WITH`, `CONNECT BY`, `WITH RECURSIVE`, `LEVEL`)
    - [[SQL_Pivot_Unpivot]] : 행을 열로, 열을 행으로 (`PIVOT`, `UNPIVOT`, `FILTER`, `UNION ALL`)

- **고급 문자열 처리 (전처리)**
    - [[SQL_Regular_Expression]] : 복잡한 문자열 패턴 찾고 발라내기 (`REGEXP_LIKE`, `REGEXP_EXTRACT`, `REGEXP_REPLACE`, `패턴 매칭`)

---

### Level 4. [SQLD 과목 2 - 제3장] 관리 구문 (Management)

> **"데이터베이스의 구조를 만들고, 데이터를 삽입/수정하며, 권한을 통제합니다."**

- **DML (데이터 조작어)**
    - [[SQL_DML_CRUD]] : 데이터 넣고, 수정하고, 지우기 (`INSERT`, `UPDATE`, `DELETE`, `MERGE`, `UPSERT`)

- **TCL (트랜잭션 제어어)**
    - [[SQL_Database_Transactions_TCL]] : 트랜잭션의 생명선 (`COMMIT`, `ROLLBACK`, `BEGIN`, `ACID`, `SAVEPOINT`, `LOCK`)

- **DDL (데이터 정의어)**
    - [[SQL_DDL_Create]] : 테이블 생성과 삭제 (`CREATE`, `DROP`, `ALTER`, `TRUNCATE`, `CONSTRAINT`, `CTAS`)

- **DCL (데이터 제어어)**
    - [[SQL_DCL_Grant_Revoke]] : 유저 권한 부여 및 회수 (`GRANT`, `REVOKE`, `ROLE`, `WITH GRANT OPTION`, `CASCADE`)

---

## Level 5. [SQLD 과목 3] SQL 고급활용 및 튜닝

> **"SQL이 내부에서 어떻게 돌아가는지 이해하고, 느린 쿼리를 빠르게 고치는 기술입니다."**

* 1장. SQL 수행 구조
    * [[SQL_Execution_Structure]] : DB 엔진이 SQL을 처리하는 방법
      (`파싱`, `최적화`, `실행`, `메모리 구조`, `버퍼 캐시`, `Sequential/Random I/O`)

    * [[SQL_Storage_Structure]] : 데이터가 디스크에 저장되는 방식
      (`블록(Block)`, `익스텐트(Extent)`, `세그먼트(Segment)`, `테이블스페이스`, `로우 체이닝/마이그레이션`)

- **2장. SQL 분석 도구**
    - [[SQL_Analysis_Tools]] : 느린 쿼리 진단하기 (`실행계획`, `EXPLAIN PLAN`, `AUTOTRACE`, `통계정보 읽는 법`)

- **3장. SQL 옵티마이저**    
    - [[SQL_Optimizer]] : 최적 실행계획을 결정하는 DB의 두뇌 (`RBO vs CBO`, `소프트/하드 파싱`, `바인드 변수`, `힌트(HINT)`)

- **4장. 인덱스 튜닝**
    - [[SQL_Index_Tuning]] : 조회 속도를 10배 올리는 핵심 기술 (`B-Tree 구조`, `인덱스 스캔 종류`, `손익분기점`, `결합 인덱스`, `인덱스 설계 원칙`)

- **5장. 조인 튜닝**
    - [[SQL_Join_Tuning]] : 조인 방식에 따른 성능 차이 (`Nested Loop Join`, `Hash Join`, `Sort Merge Join`, `드라이빙 테이블`, `조인 순서`)

- **6장. 고급 SQL 튜닝**
    - [[SQL_Advanced_Tuning]] : 소트와 DML 성능 최적화 (`불필요한 소트 제거`, `DISTINCT vs GROUP BY`, `DML 튜닝`, `파티셔닝 활용`)

- **7장. Lock과 트랜잭션 동시성 제어**
    - [[SQL_Lock_Concurrency]] : 동시에 접근할 때 데이터를 지키는 방법 (`Lock 종류`, `Deadlock`, `트랜잭션 격리 수준`, `낙관적/비관적 잠금`)

---

### 실전 연습 (Projects)

- [[00_SQL_Challenge_DashBoard|🏆 매일 SQL 챌린지 대시보드]]