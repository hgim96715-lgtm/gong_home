
>목표: SQLD 자격증 + 데이터 엔지니어 실무 SQL Oracle · PostgreSQL 병행 학습

---
---

## 문제 상황별 도구 선택 치트시트


|문제 상황|쓸 도구|핵심 노트|
|---|---|---|
|부서별 평균 / 합계 구하기|`GROUP BY` + 집계함수|[[SQL_Aggregate_GROUP_BY]]|
|원본 행 유지하면서 평균 옆에 붙이기|`OVER(PARTITION BY)`|[[SQL_Window_Functions]]|
|1등 / 상위 N명 뽑기|`RANK()` / `DENSE_RANK()`|[[SQL_Window_Functions]]|
|각 그룹에서 최신 1건만|`ROW_NUMBER()` WHERE rn=1|[[SQL_Window_Functions]]|
|이전 행 / 다음 행 값 비교|`LAG()` / `LEAD()`|[[SQL_Window_Functions]]|
|연속 구간 찾기 (Gaps & Islands)|`year - ROW_NUMBER()` + GROUP BY|[[SQL_Window_Functions]]|
|이동 평균 / 누적합|`ROWS BETWEEN`|[[SQL_Window_Functions]]|
|NULL 을 다른 값으로 대체|`COALESCE()` / `NVL()`|[[SQL_NULL_Functions]]|
|조건에 따라 값 분기|`CASE WHEN`|[[SQL_CASE_WHEN]]|
|두 테이블 연결하기|`JOIN` (INNER / LEFT / FULL)|[[SQL_Standard_JOIN]]|
|자기 자신 비교 (사원-매니저)|`Self JOIN`|[[SQL_Self_Join]]|
|있는지 없는지 확인|`EXISTS` / `IN`|[[SQL_SubQuery]]|
|쿼리 결과 위 아래로 붙이기|`UNION ALL` / `UNION`|[[SQL_UNION]]|
|복잡한 쿼리 단계별로 쪼개기|`WITH (CTE)`|[[SQL_WITH_CTE]]|
|있으면 UPDATE 없으면 INSERT|`ON CONFLICT` / `MERGE`|[[SQL_DML_CRUD]]|
|행을 열로 / 열을 행으로|`PIVOT` / `UNPIVOT`|[[SQL_Pivot_Unpivot]]|
|패턴 문자열 찾기|`LIKE` / `REGEXP`|[[SQL_Filtering_WHERE]]|
|계층 구조 (상위-하위 관계)|`WITH RECURSIVE` / `CONNECT BY`|[[SQL_Hierarchical_Query]]|
|소계 / 중계 / 총계 한 번에|`ROLLUP` / `CUBE`|[[SQL_Group_Functions]]|

---
---
## 실무 빈출 매핑

| 상황          | 실무 (PostgreSQL)                    |
| ----------- | ---------------------------------- |
| 가용병상 지역별 평균 | `AVG() OVER (PARTITION BY region)` |
| 포화 병원 순위    | `RANK() OVER (ORDER BY hvec ASC)`  |
| 최신 데이터 1건   | `DISTINCT ON (hpid)`               |
| NULL 대체     | `COALESCE(hvec, 0)`                |
| 문자열 자르기     | `SPLIT_PART(duty_addr, ' ', 2)`    |
| 있으면 UPDATE  | `ON CONFLICT DO UPDATE`            |
| 날짜 시간대      | `NOW() AT TIME ZONE 'Asia/Seoul'`  |

---
---
## Level 0. 핵심 사고력

```
쿼리를 짜기 전에 뇌를 먼저 SQL 모드로 세팅
실행 순서를 모르면 쿼리를 짜도 왜 틀렸는지 모른다
```

|노트|핵심 개념|
|---|---|
|[[01_SQL_Thinking_Roadmap]]|쿼리 짜기 전 필독 / 실행 순서 & 전략|
|[[SQL_Execution_Order]]|작성 순서 vs 실행 순서 / 치트시트|
|[[SQL_Main_Table_Strategy]]|FROM 주인공 정하는 원칙|

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

| 노트                         | 핵심 개념                                                                                               |
| -------------------------- | --------------------------------------------------------------------------------------------------- |
| [[SQL_Numeric_Functions]]  | ROUND / TRUNC / MOD / CEIL / FLOOR / LEAST / GREATEST                                               |
| [[SQL_String_Functions]]   | LTRIM / RTRIM / TRIM / SUBSTR / REPLACE / SPLIT_PART / LPAD / RPAD / LENGTH / STRING_AGG/SPLIT_PART |
| [[SQL_Date_Functions]]     | NOW / CURRENT_DATE / INTERVAL / DATE_TRUNC / EXTRACT / TO_CHAR 포맷                                   |
| [[SQL_Type_Casting]]       | CAST / TO_CHAR / TO_DATE / 명시적·암시적 형변환                                                              |
| [[SQL_Understanding_NULL]] | 3값 논리 / NULL 연산 / IS NULL                                                                           |
| [[SQL_NULL_Functions]]     | COALESCE / NVL / NULLIF / NVL2                                                                      |

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
| [[SQL_Self_Join]]          | Self Join / 사원-매니저 / 계층 비교                                 |

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

| 노트                         | 핵심 개념                                                      |
| -------------------------- | ---------------------------------------------------------- |
| [[SQL_Group_Functions]]    | ROLLUP / CUBE / GROUPING SETS / GROUPING                   |
| [[SQL_Window_Functions]]   | RANK / DENSE_RANK / ROW_NUMBER / PARTITION BY / LAG / LEAD |
| [[SQL_Top_N_Query]]        | LIMIT / ROWNUM / FETCH FIRST N ROWS ONLY                   |
| [[SQL_Hierarchical_Query]] | START WITH / CONNECT BY / WITH RECURSIVE / LEVEL           |
| [[SQL_Pivot_Unpivot]]      | PIVOT / UNPIVOT / FILTER / UNION ALL                       |

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
|[[SQL_Stored_Function]]|CREATE FUNCTION / plpgsql / RETURN QUERY / IF-THEN / OFFSET|


---

---

## PostgreSQL 실습 환경


|노트|핵심 개념|
|---|---|
|[[PostgreSQL_Setup]]|Docker 환경 세팅 / DataGrip 연결 / 권한·소유권 관리|
|[[Python_Database_Connect]]|psycopg2 / execute_values / sqlalchemy / .env|
|[[Airflow_Hooks]]|PostgresHook / execute_values UPSERT / copy_expert|


---

---

## 실전 연습

- [[00_SQL_Challenge_DashBoard]] — 매일 SQL 챌린지 대시보드

---

---
