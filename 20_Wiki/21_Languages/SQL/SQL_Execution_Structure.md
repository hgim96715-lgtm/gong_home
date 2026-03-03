---
aliases:
  - Parsing
tags:
  - SQL
related:
  - "[[SQL_Optimizer]]"
  - "[[00_SQL_HomePage]]"
---
# SQL 수행 구조 (SQL Execution Structure)

> DB 엔진이 SQL을 받아서 결과를 돌려주기까지의 내부 처리 흐름과 메모리 구조

---

## 전체 흐름 한눈에 보기

```
[클라이언트] SQL 전송
      ↓
[1단계] 파싱 (Parsing)        ← 문법 검사, 캐시 확인
      ↓
[2단계] 최적화 (Optimization)  ← 옵티마이저가 실행계획 결정
      ↓
[3단계] 실행 (Execution)       ← 실행계획대로 데이터 접근
      ↓
[4단계] Fetch                  ← 결과를 클라이언트에 전달
```

---

## 각 단계 상세

### 1단계 — 파싱 (Parsing)

SQL을 실행하기 전에 두 가지를 먼저 확인한다.

|검사 항목|내용|
|---|---|
|**문법 검사 (Syntax)**|SQL 문법이 올바른지 확인 (`SELECT`, `FROM` 순서 등)|
|**의미 검사 (Semantic)**|테이블/컬럼이 실제로 존재하는지, 권한이 있는지 확인|

검사 통과 후 **라이브러리 캐시(Library Cache)** 에서 동일 SQL을 찾는다.

```
캐시에 있음 → 소프트 파싱 (실행계획 재사용, 빠름)
캐시에 없음 → 하드 파싱 (실행계획 새로 생성, 느림)
```

### 2단계 — 최적화 (Optimization)

하드 파싱 시 옵티마이저가 실행계획을 생성한다.

```
통계 정보 수집 → 실행계획 후보 생성 → 비용 계산 → 최소 비용 계획 선택
```

> 상세 내용은 [[SQL_Optimizer]] 참고

### 3단계 — 실행 (Execution)

선택된 실행계획대로 데이터에 접근한다.

```
실행계획 확인 → 버퍼 캐시 조회
                    ↓
          ┌─────────┴──────────┐
        HIT                  MISS
   (캐시에 있음)           (캐시에 없음)
        ↓                     ↓
    바로 반환           디스크에서 읽기
                       → 캐시에 적재
                       → 데이터 처리
```

### 4단계 — Fetch

처리된 결과를 클라이언트에 전송한다.

|모드|방식|
|---|---|
|`ALL_ROWS`|전체 결과를 한 번에 전송|
|`FIRST_ROWS(n)`|앞에서부터 n건씩 전송|

---

## 메모리 구조

Oracle과 PostgreSQL 모두 **디스크 I/O를 최소화**하기 위해 메모리를 적극 활용한다.  
구조 이름은 다르지만 역할은 동일하다.

### Oracle 메모리 구조 (★★★ SQLD 핵심)

Oracle의 메모리는 크게 **SGA(공유 메모리)** 와 **PGA(프로세스 전용 메모리)** 로 나뉜다.

```
Oracle 메모리
│
├── SGA (System Global Area) ← 모든 세션이 공유하는 메모리
│   ├── 버퍼 캐시 (Buffer Cache)        ← 디스크 블록을 메모리에 캐싱
│   ├── 라이브러리 캐시 (Library Cache)  ← 실행계획 · SQL 텍스트 캐싱
│   ├── 딕셔너리 캐시 (Dictionary Cache) ← 테이블/컬럼 메타데이터 캐싱
│   └── 리두 로그 버퍼 (Redo Log Buffer) ← 변경 내역 임시 저장
│
└── PGA (Program Global Area) ← 각 세션(프로세스)이 독점 사용
    ├── 정렬 영역 (Sort Area)  ← ORDER BY, GROUP BY 처리
    ├── 해시 영역 (Hash Area)  ← Hash Join 처리
    └── 세션 정보              ← 커서, 바인드 변수 등
```

**SGA vs PGA 핵심 차이**

|구분|SGA|PGA|
|---|---|---|
|**공유 여부**|모든 세션이 **공유**|각 세션이 **독점**|
|**크기**|DB 전체에서 1개|세션 수만큼 존재|
|**주요 내용**|버퍼 캐시, 실행계획 캐시|정렬 영역, 해시 영역|
|**영향 범위**|전체 성능에 영향|해당 세션 성능에만 영향|

### PostgreSQL 메모리 구조

```
PostgreSQL 메모리
│
├── Shared Buffers           ← Oracle 버퍼 캐시와 동일 (데이터 블록 캐싱)
├── WAL Buffers              ← Oracle 리두 로그 버퍼와 동일 (변경 내역 임시 저장)
└── work_mem                 ← Oracle PGA Sort/Hash 영역과 동일 (세션별 정렬·해시 처리)
```

### Oracle vs PostgreSQL 메모리 비교

|역할|Oracle|PostgreSQL|
|---|---|---|
|데이터 블록 캐싱|Buffer Cache (SGA)|Shared Buffers|
|실행계획 캐싱|Library Cache (SGA)|내부 자동 관리|
|메타데이터 캐싱|Dictionary Cache (SGA)|System Catalog Cache|
|변경 내역 임시 저장|Redo Log Buffer (SGA)|WAL Buffers|
|정렬 · 해시 처리|Sort/Hash Area (PGA)|work_mem|
|공유 여부|SGA = 공유 / PGA = 전용|Shared Buffers = 공유 / work_mem = 세션별|

---
## 버퍼 캐시 (Buffer Cache) ★★★

**디스크에서 읽은 데이터 블록을 메모리에 올려두는 공간.**  
같은 데이터를 다시 조회할 때 디스크를 다시 읽지 않고 메모리에서 바로 가져온다.

### 논리적 읽기 vs 물리적 읽기 (★ 시험 출제)

|구분|의미|속도|Oracle|PostgreSQL|
|---|---|---|---|---|
|**논리적 읽기**|버퍼 캐시에서 블록을 읽음 (캐시 HIT)|빠름|Logical Reads|Buffer Hits|
|**물리적 읽기**|디스크에서 블록을 읽음 (캐시 MISS)|느림|Physical Reads|Block Read Time|

### Sequential I/O vs Random I/O (★ 시험 출제)

물리적 읽기(디스크 I/O)는 데이터를 읽는 **방식**에 따라 두 가지로 나뉜다.

| 구분 | Sequential I/O | Random I/O |
|---|---|---|
| **읽는 방식** | 디스크 블록을 **처음부터 순서대로** 읽음 | 필요한 블록만 **여기저기 건너뛰며** 읽음 |
| **속도** | 빠름 | 느림 |
| **발생 상황** | Full Table Scan | 인덱스를 통한 테이블 접근 |
| **I/O 단위** | 여러 블록을 한 번에 (Multiblock Read) | 한 번에 1블록씩 (Single Block Read) |

> **역설:** 인덱스를 쓰면 오히려 느려질 수 있다.  
> 인덱스 → 테이블 접근은 **Random I/O** 방식이고,  
> Full Scan은 **Sequential I/O** 방식이라 대량 데이터에선 Full Scan이 더 빠를 수 있다.

**예시로 이해하기**

```
Full Table Scan (Sequential I/O)
디스크: [블록1] → [블록2] → [블록3] → [블록4] → ... 순서대로 쭉 읽기
        한 번 이동으로 여러 블록을 연속 처리 → 빠름

인덱스 스캔 (Random I/O)
인덱스: "조건에 맞는 행은 블록7, 블록2, 블록15에 있다"
디스크: [블록7] → [블록2] → [블록15] 순서 없이 점프하며 읽기
        매번 디스크 헤드가 이동 → 느림
```

**CBO의 판단 기준**

```
조회 데이터 비율이 낮음 (소량)  →  Random I/O지만 읽는 블록 수 자체가 적음
                                    → 인덱스 스캔 선택

조회 데이터 비율이 높음 (대량)  →  Random I/O를 수백 번 반복하는 것보다
                                    Sequential I/O로 한 번에 쭉 읽는 게 유리
                                    → Full Table Scan 선택
```

> **손익분기점:** 일반적으로 전체 데이터의 **10~15%** 이상을 조회할 경우  
> 인덱스보다 Full Scan이 유리하다고 판단한다. (★ 인덱스 손익분기점과 연결)

### I/O 튜닝의 핵심 원리

```
I/O 튜닝 = Sequential I/O 비중↑ + Random I/O 발생량↓
```

| 목표 | 방법 |
|---|---|
| **Sequential I/O 비중 높이기** | 인덱스 없이 Full Scan이 유리한 구간에서는 Full Scan 유도 (`FULL` 힌트) |
| **Random I/O 발생량 줄이기** | 결합 인덱스 설계로 테이블 접근 횟수 자체를 줄임 / 클러스터링으로 관련 데이터를 물리적으로 인접 저장 |

> **핵심:** Random I/O는 건수가 많아질수록 비용이 기하급수적으로 증가한다.  
> 인덱스가 있더라도 **Random I/O 발생량이 지나치게 많아지는 구간에서는**  
> CBO가 스스로 Full Scan(Sequential I/O)을 선택하는 이유가 바로 여기에 있다.

> [!tip]
> **튜닝 목표:** 물리적 읽기↓ + 논리적 읽기 비율(캐시 히트율)↑
>
> **캐시 히트율 계산 공식**
>
> ```
> 논리적 읽기 (Logical Reads) = Query  +  Current
>                                ↑           ↑
>                          읽기 목적으로   변경 목적으로
>                          캐시에서 읽은   캐시에서 읽은
>                          블록 수         블록 수
>
> 물리적 읽기 (Physical Reads) = Disk
>                                 ↑
>                           캐시에 없어서
>                           디스크에서 읽은
>                           블록 수
>
> 캐시 히트율 = (Query + Current) / (Query + Current + Disk) × 100
> ```
>
> **95% 이상이면 양호**, 그 이하면 버퍼 캐시 크기 또는 인덱스 설계 점검 필요

Query vs Current 차이 보충:

|구분|의미|발생 상황|
|---|---|---|
|**Query**|읽기 전용으로 캐시에서 읽은 블록|SELECT 조회 시|
|**Current**|변경을 위해 캐시에서 읽은 블록|INSERT/UPDATE/DELETE 시|
|**Disk**|캐시에 없어서 디스크에서 읽은 블록|캐시 MISS 발생 시|

### 버퍼 캐시 확인 방법

```sql
-- Oracle: 실행계획 + 통계 동시 출력
SET AUTOTRACE ON STATISTICS;
SELECT * FROM emp WHERE deptno = 10;
-- 결과 하단 Statistics 항목:
-- consistent gets = 논리적 읽기 (캐시에서 읽은 블록)
-- physical reads  = 물리적 읽기 (디스크에서 읽은 블록)

-- Oracle: 캐시 히트율 전체 조회
SELECT ROUND((1 - (phyrds / (dbgets + congets))) * 100, 2) AS hit_ratio
FROM v$buffer_pool_statistics;
```

```sql
-- PostgreSQL: EXPLAIN ANALYZE + BUFFERS (블록 사용량 포함)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM emp WHERE deptno = 10;
-- 출력 예시:
-- Buffers: shared hit=5    ← 캐시에서 읽은 블록 수 (논리적 읽기)
-- Buffers: shared read=2   ← 디스크에서 읽은 블록 수 (물리적 읽기)

-- PostgreSQL: 테이블별 히트율 조회
SELECT
    relname,
    heap_blks_hit                                              AS 캐시_히트,
    heap_blks_read                                             AS 디스크_읽기,
    ROUND(heap_blks_hit::numeric /
          NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS 히트율
FROM pg_statio_user_tables
ORDER BY 히트율 ASC;
```

---

##  라이브러리 캐시 (Library Cache)

**실행계획과 SQL 텍스트를 저장해두는 공간.**  
같은 SQL이 다시 들어오면 실행계획을 새로 만들지 않고 재사용한다.

```
SQL 수신 → SQL 텍스트 해시값으로 라이브러리 캐시 조회
                ↓
    ┌───────────┴────────────┐
  캐시 HIT                캐시 MISS
  (소프트 파싱)            (하드 파싱)
  실행계획 재사용           실행계획 새로 생성
  빠름 / 부하 낮음          느림 / CPU 부하 큼
                            → 캐시에 저장
```

> **SQL 텍스트는 대소문자, 공백 하나까지 완전히 일치해야 캐시를 재사용한다.**

```sql
-- ❌ 아래 3개는 서로 다른 SQL로 인식 → 하드 파싱 3번 발생
select * from emp where deptno = 10;
SELECT * FROM emp WHERE deptno = 10;
SELECT  *  FROM emp WHERE deptno = 10;   -- 공백 차이만으로도 다른 SQL

-- ✅ 완전히 동일한 텍스트 + 바인드 변수 → 캐시 재사용
SELECT * FROM emp WHERE deptno = :deptno;
```

---

## 리두 로그 / WAL — 장애 복구의 핵심

DML(INSERT/UPDATE/DELETE) 수행 시 **변경 내역을 먼저 로그에 기록**한 후 데이터를 변경한다.  
장애 발생 시 로그를 재실행(Redo)해서 커밋된 데이터를 복구한다.

|항목|Oracle|PostgreSQL|
|---|---|---|
|**로그 이름**|Redo Log|WAL (Write-Ahead Log)|
|**메모리 버퍼**|Redo Log Buffer (SGA)|WAL Buffers|
|**역할**|트랜잭션 복구, 장애 복구|동일|
|**기록 순서**|로그 먼저 → 데이터 나중|동일 (Write-Ahead 방식)|

> **Write-Ahead Logging(WAL):** 데이터를 디스크에 쓰기 전에 **반드시 로그를 먼저 기록**한다.  
> DB가 갑자기 꺼져도 로그를 재실행해서 COMMIT된 데이터를 복구할 수 있다.

---

## 7. 핵심 정리 & 실수 주의

**"SGA와 PGA의 차이는?"**  
→ SGA = 모든 세션이 **공유**하는 메모리 (버퍼 캐시, 실행계획 캐시 등)  
→ PGA = 각 세션이 **독점**하는 메모리 (정렬, 해시 조인 처리)

**"버퍼 캐시가 크면 무조건 좋다?"**  
→ **아니다.** 너무 크면 관리 오버헤드 증가.  
→ 히트율 95% 이상이면 적정 수준이다.

**"논리적 읽기가 많으면 나쁜 쿼리다?"**  
→ **아니다.** 논리적 읽기는 캐시에서 읽는 것이므로 빠르다.  
→ 문제는 **물리적 읽기(디스크 I/O)** 가 많은 경우다.

**"SQL 텍스트가 비슷하면 캐시를 재사용한다?"**  
→ **아니다.** 대소문자, 공백 하나까지 **완전히 동일**해야 재사용한다.  
→ 바인드 변수를 쓰는 이유가 바로 이 때문이다.

**"SQLD 시험 출제 포인트는?"**  
→ **SGA vs PGA 구분**, **버퍼 캐시 / 라이브러리 캐시 역할**,  
→ **논리적 읽기 vs 물리적 읽기**, **소프트 파싱 vs 하드 파싱** 이 4가지가 핵심이다.