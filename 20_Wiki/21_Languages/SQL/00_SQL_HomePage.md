

### Level 0. 핵심 사고력 (Data Engineer's Mindset)

> **"쿼리를 짜기 전에 뇌를 먼저 SQL 모드로 세팅합니다."**

- [[01_SQL_Thinking_Roadmap]] : 🚨 쿼리 짜기 전 필독! (실행 순서 & 전략)
- [[SQL_Execution_Order]] : 작성 순서 vs 실행 순서 완벽 정리 (치트시트)
- [[SQL_Main_Table_Strategy]] : 쿼리의 주인공(FROM)을 정하는 절대 원칙

### Level 1. [SQLD 과목 1] 데이터 모델링의 이해

> **"데이터베이스의 기초 개념과 집을 짓는 설계도(ERD)를 배웁니다."**

- **1장. 데이터 모델링의 이해**
    
    - [[Concept_Database]] : DB란 무엇인가?
    - [[Concept_RDBMS]] : 관계형 데이터베이스(RDBMS)의 정체
    - [[Data_Modeling_Overview]] : 모델링의 3단계와 관점 (개념/논리/물리)
    - [[ERD_Components]] : 엔터티(Entity), 속성(Attribute), 관계(Relationship)
    - [[Keys_and_Identifiers]] : PK와 FK의 관계 (본질식별자, 인조식별자)

- **2장. 데이터 모델과 성능**
    
    - [[Normalization_Theory]] : 테이블을 쪼개는 기술, 정규화 (1NF, 2NF, 3NF)
    - [[De_Normalization]] : 성능을 위해 다시 합치는 기술, 반정규화
    - [[SQL_Denormalization_Deep_Dive]] : 반정규화 SQLD 빈출 개념 심화 파헤치기


### Level 2. [SQLD 과목 2 - 제1장] SQL 기본 (Basic)

> **"데이터를 조회하고, 가공하고, 테이블끼리 연결하는 필수 기술입니다."** (조인 편입!)

- **기초 조회 및 필터링**
    
    - [[Concept_What_is_SQL]] : SQL의 본질 (집합적 사고방식)
    - [[Concept_Table_Anatomy]] : 테이블 구조 해부 완벽 정리
    - [[SQL_SELECT_FROM]] : 데이터를 가져오는 가장 기본적인 구조
    - [[SQL_Data_Types]] : 문자, 숫자, 날짜 타입의 이해
    - [[SQL_Filtering_WHERE]] : 원하는 데이터만 쏙 골라내기
    - [[SQL_ORDER_BY]] : 데이터 줄 세우기 (ASC, DESC)

- **함수와 NULL의 이해**
    
    - [[SQL_Understanding_NULL]] : SQL의 영원한 골칫거리, NULL의 모든 것
    - [[SQL_Numeric_Functions]] : 숫자형 함수 완벽 정리
    - [[SQL_String_Functions]] : 문자열 자르고 붙이기(`LTRIM`,`RTRIM`,`REPLACE`,`SPLIT`,`TRIM`)
    - [[SQL_Date_Functions]] : 날짜 다루기 (`NOW`,`CURRENT_DATE`,`INTERVAL`)
    - [[SQL_Type_Casting]] : 명시적/암시적 형변환
    - [[SQL_NULL_Functions]] : 텅 빈 데이터(NULL) 심폐소생술 (`COALESCE` 등)

- **그룹화 및 집계**
    
    - [[SQL_Aggregate_GROUP_BY]] : 그룹별 통계 내기
    - [[SQL_DISTINCT_vs_GROUP_BY]] : 중복 제거 vs 묶어서 계산
    - [[HAVING_vs_WHERE]] : 필터링 시점의 차이
    - [[SQL_CASE_WHEN]] : IF-ELSE 로직 구현하기

- **조인 (JOIN)**

    - [[SQL_JOIN_Concept]] : 데이터 연결의 본질과 원리 (어떻게 연결할 것인가?)(`EQUI JOIN`,`NON EQUI JOIN`,`3개 이상의 다중 테이블 조인`)
    - [[SQL_Standard_JOIN]] : ANSI 표준 조인 문법 (어떤 형태로 결합할 것인가?)(`INNER JOIN`,`OUTER JOIN (LEFT, RIGHT, FULL)`,`CROSS JOIN`,`NATURAL JOIN`)


### Level 3. [SQLD 과목 2 - 제2장] SQL 활용 (Advanced)

> **"복잡한 로직을 구현하고, 데이터를 입체적으로 분석하는 고급 기술입니다."**

- **서브쿼리 및 집합 연산**
    
    - [[SQL_SubQuery]] : 복잡한 쿼리 관리 (인라인 뷰, 스칼라, 중첩 서브쿼리, 연관/비연관)
    - **[[SQL_View]] : 쿼리를 저장하는 마법의 거울 (가상 테이블과 보안 통제- 보안성,독립성,편리성)**
    - [[SQL_WITH_CTE]] : 쿼리를 변수처럼 조립하는 1회용 임시 테이블 (Top-Down 가독성 모듈화)
    - **[[SQL_UNION]] : 집합 연산자 (결과를 위아래로 이어 붙이기: UNION, INTERSECT, MINUS/EXCEPT)**

- **분석 및 고급 집계**

	- [[SQL_Group_Functions]] : 소계와 총계 내기 (`ROLLUP`, `CUBE`, `GROUPING SETS`) 
    - [[SQL_Window_Functions]] : 순위, 누적합, 이동평균의 마법 (`RANK`, `PARTITION BY`)
    - [[Top_N_Query]] : 상위 N개 데이터 추출하기 (`LIMIT`, `ROWNUM` 등) 
    -  [[Pivot_Unpivot]] : 행을 열로, 열을 행으로 (데이터 형태 변환)


### Level 4. [SQLD 과목 2 - 제3장] 관리 구문 (Management)

> **"데이터베이스의 구조를 만들고, 데이터를 삽입/수정하며, 권한을 통제합니다."** (신설 챕터!)

- **DML (데이터 조작어)**

    - [[SQL_DML_CRUD]] : 데이터 넣고, 수정하고, 지우기 (`INSERT`, `UPDATE`, `DELETE`)

- **TCL (트랜잭션 제어어)**

    - [[Database_Transactions_TCL]] : 트랜잭션의 생명선 (`COMMIT`, `ROLLBACK`)

- **DDL (데이터 정의어)**
    
    - [[SQL_DDL_Create]] : 테이블 생성과 삭제 (`CREATE`, `DROP`, `ALTER`)
    - [[SQL_Constraints]] : 이상한 데이터 차단하는 문지기 (`PK`, `FK`, `NOT NULL`)
    - [[SQL_Auto_Increment]] : 번호표 자동 발급기
    - [[SQL_Concepts_View]] : 뷰(View) - 가상 테이블과 보안

- **DCL (데이터 제어어)**

    - [[SQL_DCL_Grant_Revoke]] : 유저 권한 부여 및 회수 (`GRANT`, `REVOKE`, `ROLE`)

### 실전 연습 (Projects)

- [[00_SQL_Challenge_DashBoard|🏆 매일 SQL 챌린지 대시보드]]