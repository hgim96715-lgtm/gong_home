---
aliases:
  - Optimizer
  - RBO
  - CBO
  - NDV
  - ALL_ROWS
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
---
#  SQL 옵티마이저 (SQL Optimizer)

> SQL을 가장 빠르고 효율적으로 실행할 수 있는 최적의 실행 계획을 자동으로 결정해주는 DB 엔진의 핵심 두뇌

---

## 옵티마이저란?

사용자가 SQL을 작성할 때 **"어떻게 실행할지"는 신경 쓰지 않아도 된다.**  
옵티마이저가 자동으로 가장 효율적인 실행 경로를 찾아준다.

```
[사용자] SQL 작성 → [파서] 문법 검사 → [옵티마이저] 최적 실행계획 결정 → [실행]
```

---

##  RBO vs CBO (★★★ 시험 핵심)

| 구분 | RBO (Rule-Based Optimizer) | CBO (Cost-Based Optimizer) |
|---|---|---|
| **판단 기준** | 사전에 정의된 규칙 순위 | 예상 비용(Cost) 계산 |
| **통계 정보** | 사용 안 함 | **반드시 필요** |
| **유연성** | 낮음 (규칙이 고정) | 높음 (상황에 따라 다름) |
| **현재 사용** | 거의 사용 안 함 (구식) | **현재 표준** |
| **Oracle 지원** | 10g 이후 공식 지원 종료 | 기본값 |

### RBO — 규칙 기반

15가지 규칙에 우선순위를 매겨, 번호가 낮을수록 먼저 선택한다.

| 우선순위 | 접근 방식 |
|---|---|
| 1 | Single Row by Rowid (ROWID로 단건 접근) |
| 4 | Single Row by Unique Index (유일 인덱스로 단건) |
| 9 | Single Row by Composite Index (복합 인덱스) |
| 15 | Full Table Scan ← 최하위 |

> 인덱스가 있으면 데이터 건수와 무관하게 **무조건 인덱스를 사용**  
> → 테이블이 작아도 인덱스를 쓰는 비효율 발생

### CBO — 비용 기반

통계 정보를 바탕으로 예상 비용을 계산해 **가장 낮은 비용의 실행계획을 선택**한다.
```
예상 비용 = I/O 비용 + CPU 비용 + 메모리 비용
```

**비용(Cost)이란?**  
실제 시간(초)이 아닌 DB 내부의 **상대적인 예상 작업량 수치**. 낮을수록 효율적.

| 비용 구성 요소 | 의미 | 줄이려면 |
|---|---|---|
| **I/O 비용** | 디스크에서 블록을 읽는 횟수 | 인덱스 사용, 파티셔닝 |
| **CPU 비용** | 데이터를 처리하는 연산량 | 불필요한 정렬/함수 제거 |
| **메모리 비용** | 정렬·조인 시 필요한 메모리 | 소트 최소화, Hash Join 튜닝 |

> **I/O 비용이 가장 큰 비중** → 디스크 접근 횟수를 줄이는 것이 튜닝의 핵심

---

## CBO 통계 정보

### 통계 항목

| 통계 항목 | 설명 |
|---|---|
| `NUM_ROWS` | 테이블의 전체 행 수 |
| `BLOCKS` | 테이블이 차지하는 블록(페이지) 수 |
| `AVG_ROW_LEN` | 평균 행 길이 (바이트) |
| `NUM_DISTINCT` / **NDV** | 컬럼의 고유값 개수. 선택도 계산의 핵심 재료 |
| `DENSITY` | 컬럼 값의 분포 밀도 (선택도 계산에 사용) |
| `NUM_NULLS` | NULL 값의 개수 |

### NDV와 선택도 (Selectivity)

**NDV(Number of Distinct Values):** 컬럼에 존재하는 중복 제거된 고유값의 수.  
CBO는 NDV로 **선택도**를 계산하고, 이를 통해 예상 결과 건수(Rows)를 추정한다.

```
선택도(Selectivity) = 1 / NDV
예상 결과 건수     = NUM_ROWS × 선택도 = NUM_ROWS / NDV
```

**예시:** 사원 1000건, 부서(deptno) NDV = 5  (즉, NUM_ROWS = **1000** ← 테이블 전체 건수)
→ 선택도 = 1/5 = 0.2 → `WHERE deptno = 10` 예상 건수 = 1000 × 0.2 = **200건**

| NDV | 선택도 | 예상 건수 | 실행계획 선택 |
|---|---|---|---|
| **낮음** (고유값 적음) | 낮음 | 많음 | **Full Scan** 선택 |
| **높음** (고유값 많음) | 높음 | 적음 | **인덱스** 선택 |

### 통계 정보 수집 시 고려사항 (★ 시험 출제)

| 고려사항 | 원칙 | 잘못됐을 때 결과 |
|---|---|---|
| **시간 / 주기** | 변경 많은 테이블은 자주, 적은 테이블은 드물게. **새벽 배치 시간대**에 실행 | 오래된 통계 → 실제 분포와 달라져 잘못된 실행계획 |
| **표본 크기** | **가능한 한 적은 양의 데이터를 읽도록** 설정. 전수 조사는 정확하지만 수집 자체가 부하 유발 | 너무 작으면 통계 부정확 / 너무 크면 수집 I/O가 운영 쿼리 성능 저하 |
| **정확성** | 데이터 분포(NDV, NULL 수)를 실제와 가깝게 반영해야 올바른 비용 계산 가능 | 분포 왜곡 → 선택도 오계산 → 인덱스 무시 또는 불필요한 Full Scan |
| **안정성** | 통계가 자주 바뀌면 실행계획도 수시로 변경 → 예측 불가능한 성능 변동 | 통계 갱신 직후 갑자기 느려지는 쿼리 → 운영 장애 |
| **파티션 테이블** | 파티션별 데이터 분포가 다르므로 **파티션 단위 통계** 별도 수집 필요 | 전체를 하나로 통계 내면 분포 왜곡 → 파티션 pruning 실패 |

> **핵심:** 표본 크기는 **정확도와 수집 부하 사이의 균형점**을 찾아야 한다.  
> Oracle `AUTO_SAMPLE_SIZE` = 최소한의 데이터로 통계적으로 유의미한 결과를 내도록 자동 조정.


```sql
-- Oracle: 통계 수동 갱신 / 확인
EXEC DBMS_STATS.GATHER_TABLE_STATS('스키마명', '테이블명');
SELECT num_rows, blocks, avg_row_len, last_analyzed
FROM user_tables WHERE table_name = 'EMP';
-- last_analyzed가 오래됐다면 → 통계 갱신 필요
```
```sql
-- PostgreSQL: 통계 수동 갱신 / 확인
ANALYZE emp;
SELECT relname, n_live_tup, last_analyze
FROM pg_stat_user_tables WHERE relname = 'emp';
```

> **시험 포인트:** 데이터 100만 건인데 통계는 1000건 → CBO가 "데이터 적으니 Full Scan이 싸다" 판단  
> → **인덱스를 무시하는 잘못된 실행계획** 수립

---

##  옵티마이저 모드 (Optimizer Mode)

CBO는 "무엇을 기준으로 최적화할 것인가"에 따라 두 가지 모드로 동작한다.

| 모드 | 목표 | 특징 |
|---|---|---|
| **ALL_ROWS** | **전체 결과를 가장 빠르게** 반환 | 쿼리의 **최종 결과집합을 끝까지 Fetch하는 것을 전제**로 최적화. 처리량(Throughput) 기준. **기본값**. 배치·집계 쿼리에 유리 |
| **FIRST_ROWS(n)** | **첫 번째 n건을 가장 빠르게** 반환 | 응답속도(Response Time) 최적화. 화면 조회, 페이징에 유리 |

```sql
SELECT /*+ ALL_ROWS */       * FROM emp WHERE deptno = 10;
SELECT /*+ FIRST_ROWS(10) */ * FROM emp ORDER BY sal DESC;
```

> `ALL_ROWS` → **"끝까지 다 가져갈 것"** 전제 → Full Scan + 정렬 선호  
> `FIRST_ROWS` → **"n건만 보고 멈출 것"** 전제 → 인덱스 스캔 선호

**시험 포인트:** "CBO 기본 모드?" → **ALL_ROWS** / "페이징 처리에 유리한 모드?" → **FIRST_ROWS**

---
##  실행 계획 (Execution Plan)

옵티마이저가 SQL을 어떻게 실행할지 결정한 **로드맵**.  
실행계획을 읽으면 아래 3가지 핵심 정보를 알 수 있다.

| 실행계획에서 알 수 있는 정보 | 의미 | 예시 |
|---|---|---|
| **액세스 기법** | 테이블/인덱스에 어떻게 접근하는지 | Full Scan, Index Range Scan, Index Unique Scan 등 |
| **조인 방법** | 두 테이블을 어떤 방식으로 결합하는지 | Nested Loop, Hash Join, Sort Merge Join |
| **조인 순서** | 어느 테이블을 먼저 읽고 나중에 읽는지 | 드라이빙 테이블 → 드리븐 테이블 순서 |

### 실행계획 읽는 법

```
Id | Operation               | Name     | Rows | Cost
---|-------------------------|----------|------|-----
 0 | SELECT STATEMENT        |          |    5 |   8
 1 |  HASH JOIN              |          |    5 |   8   ← 조인 방법
 2 |   TABLE ACCESS FULL     | DEPT     |    4 |   3   ← 액세스 기법 (Full Scan)
 3 |   TABLE ACCESS BY INDEX | EMP      |   14 |   4   ← 액세스 기법 (인덱스 경유)
 4 |    INDEX RANGE SCAN     | IDX_DEPT |   14 |   2   ← 액세스 기법 (인덱스 스캔)
```

- **안쪽(들여쓰기 깊은 것)부터** 먼저 실행
- 실행 순서: `4` → `3` → `2` → `1(Hash Join)` → `0(결과 반환)`
- `Rows`: 예상 처리 건수 / `Cost`: 예상 비용 (낮을수록 좋음)

### 액세스 기법 종류

| 액세스 기법 | 설명 | Oracle 실행계획 | PostgreSQL 실행계획 |
|---|---|---|---|
| **Full Table Scan** | 테이블 전체를 처음부터 끝까지 읽음 | `TABLE ACCESS FULL` | `Seq Scan` |
| **Index Unique Scan** | PK/Unique 인덱스로 단 1건 조회 | `INDEX UNIQUE SCAN` | `Index Only Scan` |
| **Index Range Scan** | 인덱스 범위 스캔 (BETWEEN, LIKE 등) | `INDEX RANGE SCAN` | `Index Scan` |
| **Index Full Scan** | 인덱스 전체를 순서대로 스캔 | `INDEX FULL SCAN` | `Index Scan` |
| **ROWID Scan** | ROWID로 테이블 단건 직접 접근 | `TABLE ACCESS BY ROWID` | (ctid 접근) |

### 조인 방법 종류 (★ 시험 출제)

| 조인 방법 | 동작 원리 | 유리한 상황 | Oracle | PostgreSQL |
|---|---|---|---|---|
| **Nested Loop Join** | 외부 테이블 한 행마다 내부 테이블을 반복 탐색 | 소량 데이터, 인덱스 있을 때 | `NESTED LOOPS` | `Nested Loop` |
| **Hash Join** | 작은 테이블을 해시 테이블로 만들고 큰 테이블과 매칭 | 대량 데이터, 인덱스 없을 때 | `HASH JOIN` | `Hash Join` |
| **Sort Merge Join** | 두 테이블을 각각 정렬 후 병합 | 이미 정렬된 데이터, 범위 조인 | `MERGE JOIN` | `Merge Join` |

### 실행계획 확인 명령어

```sql
-- Oracle: EXPLAIN PLAN
EXPLAIN PLAN FOR
SELECT * FROM emp e, dept d WHERE e.deptno = d.deptno;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Oracle: AUTOTRACE (실행 + 실행계획 + 통계 동시 출력)
SET AUTOTRACE ON;
SELECT * FROM emp WHERE deptno = 10;
```
```sql
-- PostgreSQL: EXPLAIN (실행계획만, 실제 실행 안 함)
EXPLAIN SELECT * FROM emp e JOIN dept d ON e.deptno = d.deptno;

-- PostgreSQL: EXPLAIN ANALYZE (실제 실행 후 예상값 + 실측값 함께 출력)
EXPLAIN ANALYZE SELECT * FROM emp WHERE deptno = 10;
```

> **PostgreSQL EXPLAIN ANALYZE 출력 예시**

```
Hash Join  (cost=1.09..2.25 rows=14 width=36)
           (actual time=0.032..0.041 rows=14 loops=1)  ← 실측값
  Hash Cond: (e.deptno = d.deptno)
  -> Seq Scan on emp                                   ← Full Scan
  -> Hash
       -> Seq Scan on dept
```

---

## 힌트 (Hint) — 옵티마이저에게 직접 지시하기

CBO가 잘못된 실행계획을 선택할 때 개발자가 **강제로 방향을 지정**하는 방법.

```sql
SELECT /*+ 힌트명 */ 컬럼 FROM 테이블;
-- 일반 주석 /* */ 과 + 기호 하나 차이
```

> **PostgreSQL은 힌트 문법을 공식 지원하지 않는다.**  
> `pg_hint_plan` 확장 설치 시 Oracle과 유사한 힌트 사용 가능.  
> SQLD 시험은 **Oracle 힌트 문법 기준**으로 출제된다.

### 힌트 종류 (★ 시험 출제)

| 힌트 | 의미 |
|---|---|
| `/*+ FULL(테이블) */` | 인덱스 무시하고 **Full Table Scan** 강제 |
| `/*+ INDEX(테이블 인덱스명) */` | 지정한 **인덱스 사용** 강제 |
| `/*+ NO_INDEX(테이블 인덱스명) */` | 지정한 **인덱스 사용 금지** |
| `/*+ USE_NL(테이블) */` | **Nested Loop Join** 강제 |
| `/*+ USE_HASH(테이블) */` | **Hash Join** 강제 |
| `/*+ USE_MERGE(테이블) */` | **Sort Merge Join** 강제 |
| `/*+ LEADING(테이블) */` | 지정한 테이블을 **드라이빙 테이블**로 설정 |
| `/*+ ORDERED */` | FROM 절에 나온 **순서대로 조인** |
| `/*+ NO_MERGE */` | **View Merging 쿼리 변환이 발생하지 않도록** 막음 |
| `/*+ NO_REWRITE */` | **쿼리 재작성(Query Rewrite) 금지** |
| `/*+ NO_UNNEST */` | 서브쿼리를 **조인으로 변환(Unnesting)하지 말고 그대로 실행** |
| `/*+ NO_PUSH_PRED */` | 조인 조건을 **뷰/서브쿼리 안으로 밀어 넣지 못하도록** 막음 |

### NO_ 계열 힌트 — 옵티마이저 자동 변환 차단

| 힌트 | 옵티마이저가 하려는 것 | NO_ 힌트로 막으면 |
|---|---|---|
| `NO_MERGE` | 인라인 뷰를 메인 쿼리에 합치는 **View Merging** 수행 | View Merging 차단 → 인라인 뷰 독립 실행 |
| `NO_REWRITE` | Materialized View 등을 활용해 SQL을 빠른 형태로 재작성 | SQL 원문 그대로 실행 |
| `NO_UNNEST` | `WHERE EXISTS(서브쿼리)`를 조인으로 변환 | 서브쿼리를 조인으로 변환하지 않고 그대로 실행 |
| `NO_PUSH_PRED` | 바깥 쿼리 조인 조건을 뷰/서브쿼리 안으로 밀어 넣어 조기 필터링 | 조건을 밀지 않고 바깥에서 필터링 |

### View Merging 가능 vs 불가능 (★ 시험 단골)

| 구분 | 조건 |
|---|---|
| ✅ **가능** | 단순 SELECT + WHERE만 있는 인라인 뷰 |
| ✅ **가능** | GROUP BY 없이 단순 조인만 있는 뷰 |
| ❌ **불가능** | 뷰 안에 **GROUP BY** 가 있는 경우 |
| ❌ **불가능** | 뷰 안에 **DISTINCT** 가 있는 경우 |
| ❌ **불가능** | 뷰 안에 **집계 함수(SUM, COUNT 등)** 가 있는 경우 |
| ❌ **불가능** | 뷰 안에 **ROWNUM** 이 있는 경우 |
| ❌ **불가능** | 뷰 안에 **집합 연산자(UNION, MINUS 등)** 가 있는 경우 |
| ❌ **불가능** | **`NO_MERGE` 힌트**를 명시한 경우 |

> **암기법:** 인라인 뷰 안에서 **"행이 줄거나 묶이는 연산"** 이 있으면 View Merging 불가  
> `GROUP BY` / `DISTINCT` / `집계함수` / `ROWNUM` / `집합연산자` → 결과 구조가 바뀌므로 합치면 결과가 달라짐


```sql
-- ✅ 가능 (단순 WHERE만)
SELECT * FROM (SELECT empno, ename FROM emp WHERE sal > 3000) v
WHERE v.ename LIKE 'S%';

-- ❌ 불가능 (GROUP BY 존재)
SELECT * FROM (SELECT deptno, AVG(sal) avg_sal FROM emp GROUP BY deptno) v
WHERE v.avg_sal > 2000;

-- ❌ 불가능 (ROWNUM 존재)
SELECT * FROM (SELECT * FROM emp WHERE ROWNUM <= 5) v
WHERE v.deptno = 10;
```

---

##  옵티마이저 동작 단계 (SQL 처리 흐름)

```
① SQL 파싱 (Parsing)
   └─ 문법 오류 검사, 오브젝트 존재 여부 확인
   └─ 소프트 파싱: 라이브러리 캐시에 이미 있으면 재사용 (빠름)
   └─ 하드 파싱: 캐시에 없으면 새로 실행계획 생성 (느림, 부하 큼)

② 최적화 (Optimization)
   └─ 옵티마이저가 통계 정보 기반으로 실행계획 생성
   └─ 여러 실행계획 후보 중 최소 비용 선택

③ 실행 (Execution)
   └─ 선택된 실행계획대로 데이터 접근 및 반환
```

### 소프트 파싱 vs 하드 파싱 (★ 시험 출제)

| 구분 | 소프트 파싱 | 하드 파싱 |
|---|---|---|
| **발생 조건** | 동일 SQL이 캐시에 존재 | 캐시에 없거나 처음 실행 |
| **처리 속도** | 빠름 | 느림 |
| **시스템 부하** | 낮음 | 높음 (CPU 집중) |
| **발생 원인** | 바인드 변수 사용 시 | 리터럴 값 직접 사용 시 |

```sql
-- ❌ 하드 파싱 — 리터럴 값이 다르면 완전히 다른 SQL로 인식 → 캐시 각각 따로 점유
SELECT * FROM emp WHERE empno = 7369;
SELECT * FROM emp WHERE empno = 7499;

-- ❌ LIKE도 마찬가지
SELECT * FROM customers WHERE phone LIKE '010%';
SELECT * FROM customers WHERE phone LIKE '011%';
-- → 캐시 낭비 → 메모리 부족, 성능 저하

-- ✅ 바인드 변수 — SQL 텍스트 동일 → 캐시 1개 재사용
SELECT * FROM emp WHERE empno = :empno;
SELECT * FROM customers WHERE phone LIKE :phone_prefix;
```

**캐시 구조 비교**

```
❌ 리터럴 방식
라이브러리 캐시
├── [캐시 1] ... WHERE phone LIKE '010%'
├── [캐시 2] ... WHERE phone LIKE '011%'
└── [캐시 3] ... WHERE phone LIKE '016%'  → 3개 낭비

✅ 바인드 변수 방식
라이브러리 캐시
└── [캐시 1] ... WHERE phone LIKE :phone_prefix  → 1개 재사용
```

### 루프(Loop) 내 반복 실행 (★)

> **바인드 변수를 사용하면 루프 내 수천 번 반복 SQL도 캐시 1개만 재사용한다.**


```sql
-- ❌ 루프 + 리터럴: 10000번 하드 파싱 → 캐시 폭발
FOR i IN 1..10000 LOOP
    EXECUTE IMMEDIATE 'SELECT * FROM emp WHERE empno = ' || i;
END LOOP;

-- ✅ 루프 + 바인드 변수: 캐시 1개 10000번 재사용
FOR i IN 1..10000 LOOP
    EXECUTE IMMEDIATE 'SELECT * FROM emp WHERE empno = :empno' USING i;
END LOOP;
```

### Static SQL vs Dynamic SQL (★ 시험 출제)

| 구분 | Static SQL | Dynamic SQL |
|---|---|---|
| **SQL 확정 시점** | 프로그램 **작성(컴파일) 시점** | 프로그램 **실행 시점** |
| **SQL 변경** | 불가 | 런타임에 조건에 따라 조립 |
| **파싱 방식** | 소프트 파싱 유도 | 하드 파싱 유발 가능성 높음 |
| **SQL Injection** | 안전 | 취약 (리터럴 조립 시) |
| **캐시 효율** | 높음 | 낮음 (바인드 변수 미사용 시) |
| **사용 환경** | Pro*C, SQLJ | EXECUTE IMMEDIATE, JDBC, MyBatis |

> **핵심:** Dynamic SQL이라도 **바인드 변수로 조립**하면 소프트 파싱 유도 가능  
> "하드 파싱을 유발하는 것은?" → **리터럴 값을 직접 문자열로 붙이는 Dynamic SQL**

---

## 8. 핵심 정리 & 실수 주의

**"CBO는 항상 최선의 실행계획을 선택한다?"**  
→ **아니다.** 통계 정보가 오래되었거나 없으면 잘못된 계획 수립. 힌트로 보정 필요.

**"힌트를 쓰면 무조건 빨라진다?"**  
→ **아니다.** 잘못된 힌트는 오히려 성능 저하. 반드시 실행계획 확인 후 사용.

**"소프트 파싱을 유도하려면?"**  
→ `{text}=`, `LIKE`, `BETWEEN`, `IN` 등 조건절 종류와 무관하게 리터럴 값이 있으면 모두 하드 파싱.  
→ **바인드 변수(`:변수명`)** 사용이 유일한 해법.

**"PostgreSQL에서 힌트를 쓰려면?"**  
→ 기본적으로 미지원. `pg_hint_plan` 확장 설치 필요. SQLD 시험은 Oracle 기준 출제.