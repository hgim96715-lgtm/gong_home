---
aliases:
  - 계층형 쿼리
  - CONNECT BY
  - WITH RECURSIVE
  - 트리 구조
  - ORDER SIBLINGS BY
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Self_Join]]"
---
---

# SQL 계층형 쿼리 (Hierarchical Query)

## 개념 한 줄 요약

> **"부모-자식 관계로 연결된 트리 구조를 뿌리부터 잎까지 재귀적으로 탐색하는 쿼리."**

|사용 사례|예시|
|---|---|
|조직도|회장 → 본부장 → 팀장 → 사원|
|카테고리|의류 > 남성복 > 아우터 > 패딩|
|게시판|원글 → 답글 → 대댓글|
|제조업|부품 명세서 (BOM)|

> Self Join 은 "내 바로 위 매니저" 처럼 딱 1단계만 탐색 가능. 깊이를 알 수 없는 트리는 **계층형 쿼리**가 필요하다.

---

---

# ① 기본 문법 3대 키워드

```sql
SELECT 컬럼
FROM 테이블
START WITH 시작조건            -- 루트(최상위) 행 지정
CONNECT BY PRIOR 부모 = 자식   -- 부모 → 자식 방향 탐색
ORDER SIBLINGS BY 컬럼;        -- 형제끼리만 정렬 (트리 구조 유지)
```

|키워드|역할|
|---|---|
|`START WITH`|탐색 시작점(뿌리) 지정|
|`CONNECT BY PRIOR`|어느 컬럼에 PRIOR 를 붙이느냐로 방향 결정|
|`ORDER SIBLINGS BY`|**같은 부모를 둔 형제끼리만** 정렬|

---

## PRIOR 방향 완벽 이해

> **PRIOR = "직전에 찾은 행의 값"** PRIOR 가 붙은 쪽이 "이미 찾은 부모", 붙지 않은 쪽이 "새로 찾을 자식".

---

### 핵심 공식 

```
PRIOR 자식 = 부모  →  순방향 (위 → 아래)
PRIOR 부모 = 자식  →  역방향 (아래 → 위)
```

```text
여기서 자식/부모는 트리 위치가 아니라 컬럼 역할을 말한다.
자식 컬럼 = 자기 자신의 ID (PK)
부모 컬럼 = 부모를 가리키는 컬럼 (FK)
```

| 교재 표현           | PK/FK 표현        | 방향    |
| --------------- | --------------- | ----- |
| `PRIOR 자식 = 부모` | `PRIOR PK = FK` | 순방향 ↓ |
| `PRIOR 부모 = 자식` | `PRIOR FK = PK` | 역방향 ↑ |

### 판단 순서:

```text
① 테이블에서 어느 컬럼이 자식(PK) 이고
  어느 컬럼이 부모(FK) 인지 파악

② PRIOR 가 어느 컬럼에 붙었는지 확인

③ PRIOR 붙은 컬럼이 자식(PK) 면  →  순방향
  PRIOR 붙은 컬럼이 부모(FK) 면  →  역방향
```

---

### 예시 테이블 구조 전제

```
사원번호  =  자신의 ID        (PK)
매니저번호 =  내 부모가 누구인지  (FK)
```

|사원번호|사원명|매니저번호|
|---|---|---|
|1|회장|NULL|
|2|부장|1|
|3|과장|2|
|4|사원|3|

---

### 순방향 (위 → 아래, Top-Down)

```sql
CONNECT BY PRIOR 사원번호 = 매니저번호
```

```
문장으로 읽기:
"직전에 찾은 행의 사원번호" = "다음 행의 매니저번호"
 ↑ PRIOR = 부모 (이미 찾음)    ↑ PRIOR 없음 = 자식 (새로 찾음)

흐름: 회장(사원번호=1) → 매니저번호=1 인 부장 → 매니저번호=부장번호 인 과장 ...
결과: 회장 → 부장 → 과장 → 사원  (위에서 아래로)
```

---

### 역방향 (아래 → 위, Bottom-Up)

```sql
CONNECT BY PRIOR 매니저번호 = 사원번호
```

```
문장으로 읽기:
"직전에 찾은 행의 매니저번호" = "다음 행의 사원번호"
 ↑ PRIOR = 부모 (이미 찾음)       ↑ PRIOR 없음 = 자식 (새로 찾음)

흐름: 사원(매니저번호=3) → 사원번호=3 인 과장 → 과장의 매니저번호 = 부장 ...
결과: 사원 → 과장 → 부장 → 회장  (아래에서 위로)
```

---

### PRIOR 가 오른쪽에 있는 경우

> PRIOR 의 좌우 위치는 상관없다. **어느 컬럼에 붙었냐**만 본다.

```sql
-- 아래 두 줄은 완전히 동일한 순방향
CONNECT BY PRIOR 사원번호 = 매니저번호   -- PRIOR 왼쪽
CONNECT BY 매니저번호 = PRIOR 사원번호   -- PRIOR 오른쪽 (같은 의미)

-- 판단법:
-- PRIOR 가 붙은 컬럼이 사원번호 → 부모 = 사원번호 → 순방향
```

---
---

## ORDER BY vs ORDER SIBLINGS BY

|키워드|동작|
|---|---|
|❌ `ORDER BY`|부모-자식 관계 무시 → 트리 구조 파괴|
|✅ `ORDER SIBLINGS BY`|형제끼리만 정렬 → 트리 구조 유지|

---

---

# ② 가상 컬럼 4종 — SQLD 단골

> **가상 컬럼(Pseudo Column)** 이란 테이블에 실제로 존재하지 않지만, 특정 문법 안에서 Oracle 이 자동으로 계산해서 제공하는 읽기 전용 컬럼이다. 
> `LEVEL`, `CONNECT_BY_ISLEAF` 등은 **계층형 쿼리 안에서만** 사용할 수 있다. 일반 SELECT 에서 단독으로 쓰면 에러가 발생한다.

```sql
-- ❌ 에러: 계층형 쿼리 밖에서 사용 불가
SELECT LEVEL FROM 사원;

-- ✅ 정상: CONNECT BY 와 함께 사용
SELECT LEVEL FROM 사원
START WITH 매니저번호 IS NULL
CONNECT BY PRIOR 사원번호 = 매니저번호;
```

---

|가상 컬럼|반환값|
|---|---|
|`LEVEL`|현재 행의 깊이 (뿌리=1, 자식=2, 손자=3...)|
|`SYS_CONNECT_BY_PATH(컬럼, '/')`|뿌리부터 현재까지 경로 (예: `/회장/부장/사원`)|
|`CONNECT_BY_ROOT 컬럼`|현재 행을 파생시킨 최상위 뿌리의 값|
|`CONNECT_BY_ISLEAF`|자식 없으면 **1** (리프), 자식 있으면 **0**|

---

## 리프 데이터 (Leaf Data) 란?

> 트리 구조에서 **자식이 없는 가장 끝 노드**를 리프(Leaf, 잎) 라고 한다.

```
회장  (뿌리 - Root)
├── 부장A  (중간 노드 - Branch)
│   ├── 과장1  ← 리프 (자식 없음)
│   └── 과장2  ← 리프 (자식 없음)
└── 부장B  (중간 노드 - Branch)
    └── 과장3  ← 리프 (자식 없음)
```

```
뿌리(Root)   = 부모가 없는 최상위 노드  → START WITH 로 지정
중간(Branch) = 부모도 있고 자식도 있는 노드
리프(Leaf)   = 자식이 없는 끝 노드     → CONNECT_BY_ISLEAF = 1
```

`CONNECT_BY_ISLEAF` 활용:

```sql
SELECT
    LPAD(' ', 2*(LEVEL-1)) || 사원명  AS 조직도,
    CASE CONNECT_BY_ISLEAF
        WHEN 1 THEN '말단 사원 (리프)'
        WHEN 0 THEN '관리자 (자식 있음)'
    END AS 구분
FROM 사원
START WITH 매니저번호 IS NULL
CONNECT BY PRIOR 사원번호 = 매니저번호;
```

|사원명|CONNECT_BY_ISLEAF|구분|
|---|:-:|---|
|회장|0|관리자 (자식 있음)|
|부장A|0|관리자 (자식 있음)|
|과장1|**1**|**말단 사원 (리프)**|
|과장2|**1**|**말단 사원 (리프)**|

---

## 종합 예제

```sql
SELECT
    LEVEL                                        AS 깊이,
    LPAD(' ', 2*(LEVEL-1)) || 사원명            AS 조직도,
    CONNECT_BY_ROOT 사원명                       AS 최상위_보스,
    SYS_CONNECT_BY_PATH(사원명, ' → ')          AS 결재라인,
    CONNECT_BY_ISLEAF                            AS 리프여부
FROM 사원
START WITH 매니저번호 IS NULL
CONNECT BY PRIOR 사원번호 = 매니저번호;
```

---

---

# ③ CONNECT BY 조건 vs WHERE 조건 차이

|구분|동작 방식|
|---|---|
|`CONNECT BY` 조건|START WITH 로 확정된 루트는 **무조건 결과에 포함**. 이후 자식 전개 시 조건 적용|
|`WHERE` 조건|계층 전개가 끝난 결과셋에서 **사후에 행 제거**|

```sql
START WITH 매니저사원번호 IS NULL
CONNECT BY PRIOR 사원번호 = 매니저사원번호
       AND 입사일자 BETWEEN '2013-01-01' AND '2013-12-31'
```

> 루트(홍길동, 입사일 2012년)는 CONNECT BY 조건에 걸려도 START WITH 로 확정됐으므로 **결과에 무조건 포함된다.**

---

---

# ④ 작동 원리 추적 예제 1 — ORDER SIBLINGS BY (SQLD 기출)

## 쿼리

```sql
SELECT C3
FROM TAB1
START WITH C2 IS NULL
CONNECT BY PRIOR C1 = C2
ORDER SIBLINGS BY C3 DESC;
```

## 원본 테이블

|C1|C2|C3|
|---|---|---|
|1|NULL|A|
|2|1|B|
|3|1|C|
|4|2|D|

---

## STEP 1 — 트리 구조 파악

```
C2=NULL 인 행  →  루트
C2=1 인 행     →  C1=1 의 자식  (B, C)
C2=2 인 행     →  C1=2 의 자식  (D)
```

```
A  (C1=1)          ← 루트 (C2=NULL)
├── C  (C1=3)      ← C2=1, C3='C'
│                    └ 자식 없음 (C2=3 인 행 없음)
└── B  (C1=2)      ← C2=1, C3='B'
    └── D  (C1=4)  ← C2=2
                     └ 자식 없음 (C2=4 인 행 없음)
```

> 형제 B, C 에 ORDER SIBLINGS BY C3 DESC 적용 C3 값: B='B', C='C' → 내림차순이므로 C → B 순

---

## STEP 2 — 출력 순서 추적

```
┌──────────────────────────────────────────────────────┐
│  1번째 출력: A                                        │
│  START WITH C2 IS NULL → C1=1(A) 가 루트 확정        │
│  A 출력 후 자식 탐색: C2=1 인 행 → B, C 발견         │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  형제 B, C → ORDER SIBLINGS BY C3 DESC 적용          │
│  C3: C='C' > B='B' → C 가 먼저                       │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  2번째 출력: C                                        │
│  C(C1=3) 출력                                        │
│  C 의 자식 탐색: C2=3 인 행 없음 → 가지 끝           │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  3번째 출력: B  ← 정답                                │
│  C 가지 종료 → 다음 형제 B(C1=2) 출력                │
│  B 의 자식 탐색: C2=2 인 행 → D(C1=4) 발견           │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  4번째 출력: D                                        │
│  B 의 자식 D(C1=4) 출력                              │
│  D 의 자식 탐색: C2=4 인 행 없음 → 탐색 종료         │
└──────────────────────────────────────────────────────┘
```

## 최종 출력 순서

|순서|C3|이유|
|:-:|---|---|
|1|A|루트 (START WITH)|
|2|C|형제 중 C3 내림차순 → C 먼저|
|**3**|**B**|**C 가지 끝난 후 다음 형제**|
|4|D|B 의 자식|

> **정답: 3번째 값 = B**

```
핵심 포인트:
① ORDER SIBLINGS BY 는 형제끼리만 정렬. 전체 재정렬 아님.
② B 와 C 는 형제 → C3 DESC 기준 C → B 순.
③ D 는 B 의 자식 → B 출력 직후 이어서 나옴. 형제 정렬 대상 아님.
```

---

---

# ⑤ 작동 원리 추적 예제 2 — WHERE + CONNECT BY (SQLD 46번)

## 쿼리

```sql
SELECT COUNT(*)
FROM SQLD_46
WHERE COL1 NOT IN (3, 4)
START WITH COL1 = 1
CONNECT BY PRIOR COL1 = COL2;
```

## 원본 테이블 (SQLD_46)

|COL1|COL2|
|---|---|
|1|NULL|
|2|NULL|
|3|1|
|4|1|
|5|2|
|6|2|
|7|3|
|8|4|
|9|6|
|10|7|

---

## STEP 1 — 트리 구조 파악

```
Level 1   COL1=1
           ↓ (COL2=1 인 데이터)
Level 2   COL1=3      COL1=4
           ↓ (COL2=3, 4 인 데이터)
Level 3   COL1=7      COL1=8
           ↓ (COL2=7 인 데이터)
Level 4   COL1=10
```

---

## STEP 2 — COL2=2 인 데이터(COL1=5, 6)는 왜 포함 안 되는가?

```
COL1=2 는 COL2=NULL → 별도 루트 (COL1=1 과 무관)

START WITH COL1=1 로 시작했으므로
COL1=2 는 이 트리에 아예 없다

COL1=5 (COL2=2), COL1=6 (COL2=2) 은
COL1=2 의 자식 → COL1=2 가 트리에 없으니 포함 안 됨

COL2=2 인 데이터가 있어도 그 부모(COL1=2)가 트리 밖이면 탐색 안 됨!
```

---

## STEP 3 — 계층 전개 결과

|포함된 행|LEVEL|이유|
|---|:-:|---|
|COL1=1|1|루트 (START WITH)|
|COL1=3|2|COL2=1 → COL1=1 의 자식|
|COL1=4|2|COL2=1 → COL1=1 의 자식|
|COL1=7|3|COL2=3 → COL1=3 의 자식|
|COL1=8|3|COL2=4 → COL1=4 의 자식|
|COL1=10|4|COL2=7 → COL1=7 의 자식|

---

## STEP 4 — WHERE COL1 NOT IN (3, 4) 적용

> WHERE 는 계층 전개가 끝난 후 결과셋에서 행을 제거한다.

```
계층 전개 결과: COL1 = 1, 3, 4, 7, 8, 10  (6개)
                              ↓
            WHERE COL1 NOT IN (3, 4) 적용
            COL1=3 제거 ❌   COL1=4 제거 ❌
                              ↓
            최종 결과: COL1 = 1, 7, 8, 10  (4개)
                              ↓
                       COUNT(*) = 4
```

> **정답: ② 4**

```
핵심 포인트:
① START WITH COL1=1 → COL1=1 의 트리만 탐색.
   COL1=2 는 별도 루트 → 이 트리에 없음
   → COL2=2 인 데이터(COL1=5, 6)도 당연히 포함 안 됨

② WHERE 는 계층 전개 후 사후 제거.
   COL1=3, 4 가 제거돼도 그 자식(7, 8)은 이미 전개된 상태 → 살아있음!

③ WHERE 로 부모를 제거해도 자식은 제거되지 않는다.
   자식 전개 자체를 막으려면 CONNECT BY 조건으로 걸어야 함.
```

---

---

# ⑥ UNION 으로 위아래 동시 탐색

```sql
SELECT A.부서코드, A.부서명, A.상위부서코드, LVL
FROM (
    -- [1] 역방향: 120번 → 상위로 올라간다
    SELECT 부서코드, 부서명, 상위부서코드, LEVEL AS LVL
    FROM 부서
    START WITH 부서코드 = '120'
    CONNECT BY PRIOR 상위부서코드 = 부서코드

    UNION  -- 120번 자신 중복 제거

    -- [2] 순방향: 120번 → 하위로 내려간다
    SELECT 부서코드, 부서명, 상위부서코드, LEVEL AS LVL
    FROM 부서
    START WITH 부서코드 = '120'
    CONNECT BY 상위부서코드 = PRIOR 부서코드
) A
ORDER BY A.부서코드;
```

---

---

# ⑦ PostgreSQL — WITH RECURSIVE

```sql
WITH RECURSIVE 조직도 AS (

    -- Anchor: Oracle 의 START WITH 역할
    SELECT
        사원번호, 사원명, 매니저번호,
        1            AS 계층_깊이,
        사원명::text AS 결재라인
    FROM 사원
    WHERE 매니저번호 IS NULL

    UNION ALL

    -- Recursive: Oracle 의 CONNECT BY 역할
    SELECT
        c.사원번호, c.사원명, c.매니저번호,
        p.계층_깊이 + 1,
        p.결재라인 || ' → ' || c.사원명
    FROM 사원 c
    JOIN 조직도 p ON c.매니저번호 = p.사원번호
)
SELECT * FROM 조직도;
```

```
작동 순서:
① Anchor 실행 → 루트(매니저번호 IS NULL) 삽입 (깊이: 1)
② 1차 반복   → 루트의 자식들 추가 (깊이: 2)
③ 2차 반복   → 자식의 자식들 추가 (깊이: 3)
④ 종료       → 더 이상 JOIN 결과 없으면 자동 종료
```

## Oracle vs PostgreSQL 비교

|Oracle|PostgreSQL|
|---|---|
|`START WITH 조건`|Anchor 의 `WHERE 조건`|
|`CONNECT BY PRIOR A = B`|Recursive 의 `JOIN ON` 조건|
|`LEVEL`|Anchor 에서 `1` 초기화 후 `+1` 누적|
|`SYS_CONNECT_BY_PATH(col,'/')`|`|
|`CONNECT_BY_ROOT col`|Anchor 에서 별도 컬럼으로 전달|
|`CONNECT_BY_ISLEAF`|`NOT EXISTS(자식 쿼리)` 로 구현|
|`ORDER SIBLINGS BY`|미지원 → 직접 정렬 로직 구성|

---

---

# 초보자 실수 정리

|실수|원인|해결|
|---|---|---|
|PRIOR 방향이 헷갈림|`=` 좌우 위치가 아니라 어느 컬럼에 붙었냐가 방향 결정|`PRIOR 사원번호` = 순방향|
|ORDER BY 썼더니 조직도 깨짐|전체 재정렬 → 트리 파괴|`ORDER SIBLINGS BY` 사용|
|CONNECT BY 조건 걸었는데 루트 포함됨|START WITH 루트는 항상 포함|루트 제외 시 WHERE 별도 사용|
|WHERE 로 부모 제거했는데 자식이 살아있음|WHERE 는 전개 후 사후 제거|CONNECT BY 조건으로 전개 자체를 막기|

---
