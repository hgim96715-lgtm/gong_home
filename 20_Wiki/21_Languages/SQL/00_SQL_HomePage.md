
>목표: SQLD 자격증 + 데이터 엔지니어 실무 SQL Oracle · PostgreSQL 병행 학습

---

---

## Level 0. 핵심 사고력

```
쿼리를 짜기 전에 뇌를 먼저 SQL 모드로 세팅
실행 순서를 모르면 쿼리를 짜도 왜 틀렸는지 모른다
```

| 노트                          | 핵심 개념                   |
| --------------------------- | ----------------------- |
| [[SQL_Execution_Order]]     | 작성 순서 vs 실행 순서 / 치트시트   |
| [[SQL_Main_Table_Strategy]] | FROM 주인공 정하는 원칙         |

---

---

## Level 1. 데이터 모델링

```
DB 를 만들기 전에 설계도(ERD)를 그리는 법
SQLD 1과목 핵심 영역
```

### 데이터 모델링의 이해

|노트|핵심 개념|
|---|---|
|[[Concept_Database]]|DB란 무엇인가 / DBMS / 스키마 / 인스턴스|
|[[Concept_RDBMS]]|관계형 DB / 테이블 / 행 / 열 / 무결성|
|[[SQL_Relational_Operators]]|SELECT(σ) / PROJECT(π) / JOIN(⋈) / DIVIDE(÷)|
|[[Data_Modeling_Overview]]|개념·논리·물리 모델링 3단계|
|[[SQL_ERD_Components]]|Entity / Attribute / Relationship / 카디널리티|
|[[SQL_Keys_and_Identifiers]]|PK / FK / 본질식별자 / 인조식별자 / 복합식별자|

### 데이터 모델과 성능

|노트|핵심 개념|
|---|---|
|[[SQL_Normalization_정규화]]|이상현상 / 1NF / 2NF / 3NF / 성능 데이터 모델링|
|[[SQL_De_Normalization_반정규화]]|테이블 병합·분할·추가 / 컬럼 반정규화 / 파티셔닝|

---

---

## Level 2. SQL 기본

```
데이터를 조회하고, 가공하고, 테이블끼리 연결하는 필수 기술
```

### 기초 조회 및 필터링

|노트|핵심 개념|
|---|---|
|[[Concept_What_is_SQL]]|집합적 사고방식 / 선언형 언어 / DDL·DML·DCL·TCL|
|[[Concept_Table_Anatomy]]|행(Row) / 열(Column) / 도메인|
|[[SQL_SELECT_FROM]]|SELECT / FROM / 별칭(AS) / *|
|[[SQL_Data_Types]]|VARCHAR / NUMBER / DATE / BOOLEAN / NULL|
|[[SQL_Filtering_WHERE]]|BETWEEN / IN / LIKE / ILIKE / IS NULL / AND·OR|
|[[SQL_ORDER_BY]]|ASC / DESC / NULL 정렬 / 다중 정렬|

### 함수와 NULL

| 노트                         | 핵심 개념                                                                                    |
| -------------------------- | ---------------------------------------------------------------------------------------- |
| [[SQL_Numeric_Functions]]  | ROUND / TRUNC / MOD / CEIL / FLOOR / LEAST / GREATEST                                    |
| [[SQL_String_Functions]]   | LTRIM / RTRIM / TRIM / SUBSTR / REPLACE / SPLIT_PART / LPAD / RPAD / LENGTH / STRING_AGG |
| [[SQL_Date_Functions]]     | NOW / CURRENT_DATE / INTERVAL / DATE_TRUNC / EXTRACT / TO_CHAR 포맷                        |
| [[SQL_Type_Casting]]       | CAST / TO_CHAR / TO_DATE / 명시적·암시적 형변환                                                   |
| [[SQL_Understanding_NULL]] | 3값 논리 / NULL 연산 / IS NULL                                                                |
| [[SQL_NULL_Functions]]     | COALESCE / NVL / NULLIF / NVL2                                                           |

### 그룹화 및 집계

|노트|핵심 개념|
|---|---|
|[[SQL_Aggregate_GROUP_BY]]|COUNT / SUM / AVG / MAX / MIN / GROUP BY|
|[[SQL_DISTINCT_vs_GROUP_BY]]|DISTINCT / GROUP BY 차이|
|[[HAVING_vs_WHERE]]|WHERE → 집계 전 / HAVING → 집계 후|
|[[SQL_CASE_WHEN]]|CASE WHEN / THEN / ELSE / END / DECODE|
|[[SQL_Statistical_Functions]]|PERCENTILE_CONT / STDDEV / VARIANCE / CORR|

### JOIN

|노트|핵심 개념|
|---|---|
|[[SQL_JOIN_Concept]]|EQUI JOIN / NON EQUI JOIN / 다중 조인 원리|
|[[SQL_Standard_JOIN]]|INNER / LEFT·RIGHT·FULL OUTER / CROSS / NATURAL JOIN|

---

---

## Level 3. SQL 활용

```
복잡한 로직 구현 + 데이터 입체 분석 고급 기술
```

### 서브쿼리 & 집합 연산

|노트|핵심 개념|
|---|---|
|[[SQL_SubQuery]]|인라인 뷰 / 스칼라 / 중첩 서브쿼리 / EXISTS / IN|
|[[SQL_View]]|가상 테이블 / 보안성 / 독립성 / 편리성|
|[[SQL_WITH_CTE]]|WITH / CTE / 재귀 CTE / 가독성|
|[[SQL_UNION]]|UNION / UNION ALL / INTERSECT / MINUS·EXCEPT|

### 분석 & 고급 집계

|노트|핵심 개념|
|---|---|
|[[SQL_Group_Functions]]|ROLLUP / CUBE / GROUPING SETS / GROUPING|
|[[SQL_Window_Functions]]|RANK / DENSE_RANK / ROW_NUMBER / PARTITION BY / LAG / LEAD|
|[[SQL_Top_N_Query]]|LIMIT / ROWNUM / FETCH FIRST N ROWS ONLY|
|[[SQL_Self_Join]]|Self Join / 사원-매니저 / 계층 비교|
|[[SQL_Hierarchical_Query]]|START WITH / CONNECT BY / WITH RECURSIVE / LEVEL|
|[[SQL_Pivot_Unpivot]]|PIVOT / UNPIVOT / FILTER / UNION ALL|

### 고급 문자열 처리

|노트|핵심 개념|
|---|---|
|[[SQL_Regular_Expression]]|REGEXP_LIKE / REGEXP_EXTRACT / REGEXP_REPLACE / 패턴 매칭|

---

---

## Level 4. 관리 구문

```
DB 구조 만들기 / 데이터 삽입·수정 / 권한 통제
```

|노트|핵심 개념|
|---|---|
|[[SQL_DML_CRUD]]|INSERT / UPDATE / DELETE / MERGE / UPSERT|
|[[SQL_Database_Transactions_TCL]]|COMMIT / ROLLBACK / BEGIN / ACID / SAVEPOINT / LOCK|
|[[SQL_DDL_Create]]|CREATE / DROP / ALTER / TRUNCATE / CONSTRAINT / CTAS|
|[[SQL_DCL_Grant_Revoke]]|GRANT / REVOKE / ROLE / WITH GRANT OPTION / CASCADE|

---

---

## PostgreSQL 실습 환경

|노트|핵심 개념|
|---|---|
|[[PostgreSQL_Setup]]|Docker 환경 세팅 / DataGrip 연결 / 권한·소유권 관리|

---

---

## 실전 연습

- [[00_SQL_Challenge_DashBoard]] — 매일 SQL 챌린지 대시보드

---

---
