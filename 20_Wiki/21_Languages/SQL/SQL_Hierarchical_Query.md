---
aliases:
  - 계층형 쿼리
  - CONNECT BY
  - WITH RECURSIVE
  - 트리 구조
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Self_Join]]"
---
# SQL 계층형 쿼리 (Hierarchical Query)

> 부모-자식 관계로 이루어진 트리(Tree) 구조를 뿌리(Root)부터 잎(Leaf)까지 재귀적으로 탐색하는 쿼리 기법

---

## 왜 필요한가?

`Self Join`은 "내 바로 위 매니저"처럼 **딱 1단계**만 탐색할 수 있다.

하지만 아래처럼 깊이를 알 수 없는 구조를 탐색하려면 **재귀적으로 반복** 하는 계층형 쿼리가 필요하다.

|사용 사례|예시|
|---|---|
|조직도|회장 → 본부장 → 팀장 → 사원|
|카테고리|의류 > 남성복 > 아우터 > 패딩|
|게시판|원글 → 답글 → 답글의 답글|
|제조업|부품 명세서 (BOM)|

---

## Oracle 문법 (`START WITH ~ CONNECT BY`)

> SQLD 시험에서는 **Oracle 전용 문법**이 출제된다. 실무(PostgreSQL 등)에서는 `WITH RECURSIVE`를 사용한다.

### 기본 구조 3대 키워드

|키워드|역할|
|---|---|
|`START WITH`|탐색 **시작점(뿌리)** 지정|
|`CONNECT BY`|어느 가지를 타고 전개할지 **조인 조건** 지정|
|`PRIOR`|**"직전(이전) 단계"** 를 가리키는 방향키 — 어느 컬럼에 붙느냐에 따라 방향 결정|

---

###  PRIOR 방향 완벽 이해 (핵심 ★★★)

`PRIOR`가 **어느 컬럼에 붙느냐**에 따라 탐색 방향이 달라진다.

#### ▶ 순방향 (Top-Down: 위 → 아래)

```sql
CONNECT BY PRIOR 사원번호 = 매니저번호
-- 또는
CONNECT BY 매니저번호 = PRIOR 사원번호
```

- 해석: "직전에 찾은 사람의 **사번** = 다음 사람의 **매니저번호**"
- 결과: 회장 → 부장 → 과장 → 사원 순으로 내려간다.

#### ▶ 역방향 (Bottom-Up: 아래 → 위)

```sql
CONNECT BY PRIOR 매니저번호 = 사원번호
-- 또는
CONNECT BY 사원번호 = PRIOR 매니저번호
```

- 해석: "직전에 찾은 사람의 **매니저번호** = 다음 사람의 **사번**"
- 결과: 말단 사원 → 팀장 → 부장 → 회장 순으로 올라간다.

---

### 가상 컬럼 4종 (SQLD 단골 ★)

|가상 컬럼|반환값|
|---|---|
|`LEVEL`|현재 행의 깊이 (뿌리=1, 자식=2, 손자=3 …)|
|`SYS_CONNECT_BY_PATH(컬럼, '구분자')`|뿌리부터 현재까지의 경로 문자열 (예: `/회장/부장/사원`)|
|`CONNECT_BY_ROOT 컬럼명`|현재 행을 파생시킨 **최상위 뿌리** 의 값|
|`CONNECT_BY_ISLEAF`|자식이 없는 잎사귀이면 **1**, 자식이 있으면 **0**|

#### 종합 예제

```sql
SELECT 
    LEVEL                                          AS 계층_깊이,
    LPAD(' ', 2 * (LEVEL - 1)) || 사원명          AS 조직도,   -- 들여쓰기로 시각화
    CONNECT_BY_ROOT 사원명                         AS 최상위_보스,
    SYS_CONNECT_BY_PATH(사원명, ' → ')            AS 결재라인,  -- 예: → 회장 → 부장 → 사원
    CONNECT_BY_ISLEAF                              AS 막내여부
FROM 사원
START WITH 매니저번호 IS NULL                      -- 매니저 없는 최상위자부터 시작
CONNECT BY PRIOR 사원번호 = 매니저번호;            -- 순방향(Top-Down)
```

---

### CONNECT BY 조건 vs WHERE 조건의 차이 (★ 중요)

**CONNECT BY 절에 작성된 조건**은 **WHERE 절에 작성된 조건과 다르게 동작**한다.

|구분|동작 방식|
|---|---|
|**CONNECT BY 조건**|`START WITH`로 확정된 루트 행은 **결과에 반드시 포함**되며, 이후 자식 행을 전개할 때 조건이 적용된다.|
|**WHERE 조건**|계층 전개가 모두 끝난 결과셋에서 조건에 맞지 않는 행을 **사후에 제거**한다.|

#### 예제 — 입사일 조건을 CONNECT BY에 추가

```sql
SELECT 사원번호, 사원명, 입사일자, 매니저사원번호
FROM 사원
START WITH 매니저사원번호 IS NULL
CONNECT BY PRIOR 사원번호 = 매니저사원번호
       AND 입사일자 BETWEEN '2013-01-01' AND '2013-12-31'
ORDER SIBLINGS BY 사원번호;
```

**원본 데이터**

|사원번호|사원명|입사일자|매니저사원번호|
|---|---|---|---|
|001|홍길동|2012-01-01|NULL|
|003|이순신|2013-01-01|001|
|004|이민정|2013-01-01|001|
|005|이병헌|2013-01-01|NULL|

**실행 결과**

|사원번호|사원명|입사일자|매니저사원번호|
|---|---|---|---|
|001|홍길동|2012-01-01|NULL|
|003|이순신|2013-01-01|001|
|004|이민정|2013-01-01|001|
|005|이병헌|2013-01-01|NULL|

> ① `START WITH 매니저사원번호 IS NULL` 에 의해 001(홍길동), 005(이병헌)는 루트로 확정되어 **결과에 무조건 포함**된다.  
> ② 이후 `CONNECT BY` 조건의 입사일 범위(2013년)에 해당하는 자식들—003(이순신), 004(이민정)—이 전개되어 포함된다.  
> ⇒ 루트인 001은 입사일이 2012년이지만 `START WITH`로 이미 확정되었으므로 **필터링되지 않고 결과에 포함**된다.

---

###  정렬: `ORDER SIBLINGS BY` (★ SQLD 출제)

계층 구조 안에서 정렬이 필요할 때 일반 `ORDER BY`를 쓰면 트리 구조가 붕괴된다.

|키워드|동작|
|---|---|
|❌ `ORDER BY`|부모-자식 관계를 무시하고 전체를 단순 정렬 → **트리 구조 파괴**|
|✅ `ORDER SIBLINGS BY`|**같은 부모를 가진 형제들끼리만** 정렬 → 트리 구조 유지|

#### 예제

```sql
SELECT 
    LEVEL,
    LPAD(' ', 2 * (LEVEL - 1)) || 사원명  AS 조직도,
    급여
FROM 사원
START WITH 매니저번호 IS NULL
CONNECT BY PRIOR 사원번호 = 매니저번호
ORDER SIBLINGS BY 급여 DESC;   -- 같은 라인(형제)끼리만 급여 높은 순
```

#### 정렬 방식 비교

|구분|결과|
|---|---|
|**정렬 전**|회장(9000) → 김부장(5000) → 박과장(4000) → 이대리(3000) → 최부장(6000) → 정대리(3500)|
|❌ `ORDER BY 급여 DESC`|회장(9000) → 최부장(6000) → 김부장(5000) → 박과장(4000) → 정대리(3500) → 이대리(3000) ← 트리 붕괴|
|✅ `ORDER SIBLINGS BY 급여 DESC`|회장(9000) → 최부장(6000) → 정대리(3500) → 김부장(5000) → 박과장(4000) → 이대리(3000) ← 트리 유지|

---

### 작동 원리 추적 예제 (SQLD 기출)

```sql
SELECT C3
FROM TAB1
START WITH C2 IS NULL
CONNECT BY PRIOR C1 = C2
ORDER SIBLINGS BY C3 DESC;
```

**원본 테이블 (TAB1)**

| C1  | C2   | C3  |
| --- | ---- | --- |
| 1   | NULL | A   |
| 2   | 1    | B   |
| 3   | 1    | C   |
| 4   | 2    | D   |

**트리 구조 시각화**

```text
A  (C1=1, 루트: C2 IS NULL)
├── C  (C1=3, C2=1 → 1번의 자식) ← ORDER SIBLINGS BY C3 DESC → C가 B보다 먼저
└── B  (C1=2, C2=1 → 1번의 자식)
    └── D  (C1=4, C2=2 → 2번의 자식)
```

**단계별 실행 원리**

|출력 순서|C3|원리|
|---|---|---|
|**1번째**|**A**|`START WITH C2 IS NULL` → C1=1(A)이 루트로 확정되어 가장 먼저 출력. 자식 탐색 시작: C2=1인 행 → B(C1=2), C(C1=3) 두 개 발견|
|**2번째**|**C**|💥 **[SIBLINGS BY 작동]** B와 C는 같은 부모(C1=1)를 둔 **형제**. `C3 DESC` 정렬이므로 C > B 순 → C가 먼저 출력. C(C1=3)의 자식: C2=3인 행 없음 → 가지 끝|
|**3번째**|**B**|C 라인이 끝났으니 다음 형제 B(C1=2) 출력. B의 자식 탐색: C2=2인 행 → D(C1=4) 발견|
|**4번째**|**D**|B의 자식 D(C1=4) 출력. D의 자식: C2=4인 행 없음 → 가지 끝 → 탐색 종료|

> **정답: 3번째 표시되는 값은 ② B**

**핵심 포인트 정리**

- `ORDER SIBLINGS BY`는 **전체 재정렬이 아니라**, 같은 부모를 둔 형제끼리만 정렬한다.
- B와 C는 형제이므로 C3 DESC 기준으로 C → B 순이 된다.
- D는 B의 자식이므로, B가 출력된 직후에 이어서 나온다 (D는 형제 정렬 대상이 아님).

---

## SQLD 기출 — UNION으로 위아래 동시 탐색

특정 부서(120번)를 기준으로 **상위(역방향)** 와 **하위(순방향)** 조직도를 함께 가져오는 패턴이다.

```sql
SELECT A.부서코드, A.부서명, A.상위부서코드, B.매출액, LVL
FROM (
    -- [1] 역방향: 120번 부서 → 상위 부서로 올라간다
    SELECT 부서코드, 부서명, 상위부서코드, LEVEL AS LVL
    FROM 부서
    START WITH 부서코드 = '120'
    CONNECT BY PRIOR 상위부서코드 = 부서코드

    UNION  -- 120번 자신은 중복 제거하여 한 번만 포함

    -- [2] 순방향: 120번 부서 → 하위 부서로 내려간다
    SELECT 부서코드, 부서명, 상위부서코드, LEVEL AS LVL
    FROM 부서
    START WITH 부서코드 = '120'
    CONNECT BY 상위부서코드 = PRIOR 부서코드
) A
LEFT OUTER JOIN 매출 B ON A.부서코드 = B.부서코드
ORDER BY A.부서코드;
```

---

##  실무 문법 — PostgreSQL `WITH RECURSIVE`

Oracle의 전용 키워드 없이, **CTE(Common Table Expression)** 를 재귀 호출하는 방식이다.  
Oracle의 가상 컬럼은 직접 연산으로 구현한다.

```sql
WITH RECURSIVE 조직도 AS (

    -- [1] Anchor (시작점) — Oracle의 START WITH 역할
    SELECT 
        사원번호, 사원명, 매니저번호,
        1           AS 계층_깊이,     -- LEVEL 구현
        사원명::text AS 결재라인       -- SYS_CONNECT_BY_PATH 구현 (초기값)
    FROM 사원
    WHERE 매니저번호 IS NULL

    UNION ALL

    -- [2] Recursive (반복부) — Oracle의 CONNECT BY 역할
    SELECT 
        c.사원번호, c.사원명, c.매니저번호,
        p.계층_깊이 + 1,                      -- 부모 깊이 + 1
        p.결재라인 || ' → ' || c.사원명        -- 경로 누적
    FROM 사원 c
    JOIN 조직도 p ON c.매니저번호 = p.사원번호  -- 스스로를 재귀 JOIN

)
SELECT * FROM 조직도;
```

**작동 순서:**

1. **Anchor 실행** — `WHERE 매니저번호 IS NULL` → 회장 1명을 `조직도` 테이블에 삽입 (깊이: 1)
2. **1차 반복** — 회장을 매니저로 두는 부장들을 찾아 추가 (깊이: 2)
3. **2차 반복** — 부장들을 매니저로 두는 과장/대리들을 찾아 추가 (깊이: 3)
4. **종료** — 더 이상 조인 결과가 없으면 자동 종료, 최종 결과 반환

---

## Oracle vs PostgreSQL 비교

|Oracle (`CONNECT BY`)|PostgreSQL (`WITH RECURSIVE`)|
|---|---|
|`START WITH 조건`|Anchor의 `WHERE 조건`|
|`CONNECT BY PRIOR A = B`|Recursive의 `JOIN ON` 조건|
|`LEVEL`|Anchor에서 `1` 초기화 후 `+1` 누적|
|`SYS_CONNECT_BY_PATH(col, '/')`|문자열 연결 (`\|`)로 직접 구현|
|`CONNECT_BY_ROOT col`|Anchor에서 별도 컬럼으로 전달|
|`CONNECT_BY_ISLEAF`|`NOT EXISTS (자식 쿼리)`로 직접 구현|
|`ORDER SIBLINGS BY`|미지원 → 직접 정렬 로직 구성 필요|

---

## 초보자 실수 정리

**Q. PRIOR 위치가 헷갈려요.**  
→ `PRIOR`는 `{text}=` 기호의 좌우 위치가 아니라 **"어느 컬럼에 붙었냐"** 가 방향을 결정한다.  
→ `PRIOR 사원번호` = "직전에 찾은 사람의 사번" → 순방향(Top-Down)  
→ `PRIOR 매니저번호` = "직전에 찾은 사람의 매니저번호" → 역방향(Bottom-Up)

**Q. ORDER BY를 썼더니 조직도가 깨졌어요.**  
→ 계층 쿼리 내 정렬은 반드시 `ORDER SIBLINGS BY`를 사용해야 한다.

**Q. CONNECT BY 조건에 입사일 필터를 걸었는데 루트 행이 왜 포함되죠?**  
→ `START WITH`로 확정된 루트는 CONNECT BY 조건과 무관하게 **항상 결과에 포함**된다.  
→ 루트 자체를 제외하려면 `WHERE` 절을 별도로 사용해야 한다.