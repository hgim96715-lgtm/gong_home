---
aliases:
  - Top N Query
  - LIMIT
  - ROWNUM
  - FETCH FIRST
  - 상위 데이터 추출
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_ORDER_BY]]"
  - "[[SQL_SubQuery]]"
  - "[[SQL_Window_Functions]]"
---
# 상위 N개 데이터 추출하기 (`LIMIT`, `ROWNUM` 등)

## Concept Summary(한줄 요약)

수십만 건의 데이터 중 정렬 기준에 따라 **"가장 높은(또는 낮은) 순위의 N개 행"** 만 쏙 뽑아내는 쿼리 작성 기법

---
## Why is it needed(왜 필요한가)?

"가장 비싼 상품 3개", "최근 가입한 유저 5명"을 찾을 때, 굳이 테이블 전체 데이터를 다 불러올 필요가 없잖아? DB 서버의 메모리와 네트워크 부하를 줄이고, 웹/앱에서 게시판 페이징(1페이지당 10개씩 보여주기) 처리를 하려면 반드시 필요

---
## Practical Context(실무맥락 )

실무에서는 우리가 쓰는 **PostgreSQL**의 `LIMIT` 구문으로 너무나도 쉽게 해결하지만, **SQLD 시험**에서는 악명 높은 Oracle의 `ROWNUM` 작동 원리를 집요하게 물어보기 때문에 두 가지 방식을 모두 명확하게 구분해서 알아둬야 해.

---
## Code Core Points (DB별 Top-N 추출법)

### [PostgreSQL / MySQL] `LIMIT` 

가장 직관적이고 쉬워. 쿼리 맨 마지막에 `LIMIT N`만 딱 붙여주면 끝이야.

```sql
-- 실무(PostgreSQL): 가장 급여가 높은 3명 추출
SELECT 사원명, 급여
FROM 사원
ORDER BY 급여 DESC
LIMIT 3; 

-- 💡 참고: 3등부터 5등까지 뽑고 싶다면 OFFSET을 쓴다. (게시판 페이징 핵심)
-- LIMIT 3 OFFSET 2; (앞에 2개 건너뛰고 그다음 3개 가져와라)
```

### [Oracle / SQLD 시험용] `ROWNUM` (인라인 뷰 필수)

Oracle에는 예전에 `LIMIT`이 없었어. 대신 행이 출력되는 순서대로 1, 2, 3... 가짜 번호표를 붙여주는 `ROWNUM`이라는 가상 컬럼을 썼지.

#### ROWNUM의 절대 규칙: 건너뛰기 불가!

`ROWNUM`은 은행 창구 대기표와 같아서 1번부터 차례대로 발급되어야만 다음 번호가 존재할 수 있다.
- 항상 `<` 나 `<=` 조건으로만 사용해야 한다.
- `WHERE ROWNUM = 5`나 `ROWNUM > 3` 같은 건너뛰기 조건은 영원히 성립할 수 없다. (데이터 0건 반환)

```sql
-- ❌ [초보자의 대참사] 이렇게 짜면 절대 안 됨!!
SELECT 사원명, 급여
FROM 사원
WHERE ROWNUM <= 3    -- "아무나 3명 먼저 뽑고" (정렬되기도 전에 번호표가 붙음)
ORDER BY 급여 DESC;  -- "그 3명 안에서 정렬해라" -> 내가 원한 최고 연봉자 3명이 아님!

-- ⭕️ [정답] 인라인 뷰(서브쿼리)로 정렬부터 시키고 번호표를 잘라야 함!
SELECT *
FROM (
    SELECT 사원명, 급여
    FROM 사원
    ORDER BY 급여 DESC  -- 1. 일단 안쪽에서 완벽하게 줄부터 세운다.
)
WHERE ROWNUM <= 3;      -- 2. 바깥쪽에서 위에서부터 3명만 싹둑 자른다.
```

(참고: 최근 Oracle 12c 버전부터는 `FETCH FIRST 3 ROWS ONLY`라는 표준 문법이 생겨서 PostgreSQL의 `LIMIT`처럼 쿼리 맨 끝에 붙여서 쉽게 처리할 수도 있어!)

### 순위 윈도우 함수 활용 (`ROW_NUMBER`, `RANK`, `DENSE_RANK`)

어떤 DB(PostgreSQL, Oracle 등)를 쓰든 다 통하는 실무 표준 방식
특히 **"부서별 연봉 탑 3"** 처럼 파티션별 Top-N을 구하거나, **"공동 순위(동점자)"** 를 어떻게 자를지 디테일하게 결정해야 할 때 독보적으로 강력해.

- **`ROW_NUMBER()`**: 동점자고 뭐고 상관없이, 무조건 내 눈에 보이는 **딱 3명**만 자르고 싶을 때 (1, 2, 3)
- **`RANK()`**: 공동 2등이 2명이면 다음은 4등이 됨. `<= 3`으로 자르면 **총 3명**이 나옴 (1, 2, 2 까지만 출력되고 4등은 버려짐)
- **`DENSE_RANK()`**: 공동 2등이 2명이어도 다음 순위를 3등으로 쳐줌. `<= 3`으로 자르면 **총 4명 이상**이 나올 수도 있음! (1, 2, 2, 3 모두 출력)

>🔗 **자세한 함수별 차이는 [[SQL_Window_Functions#Code Core Points (1. 순위 함수)|순위함수]] 참고**

```sql
-- 급여 순위 번호표를 붙인 뒤, 바깥에서 3등 이하만 필터링!
WITH RankedEmp AS (  -- 💡 WITH절(CTE)이나 인라인 뷰로 한 번 감싸주는 게 핵심!
    SELECT 
        사원명, 
        급여,
        
        -- [택 1] 목적에 맞는 함수를 골라 쓴다.
        -- ROW_NUMBER() OVER(ORDER BY 급여 DESC) AS rn  -- 얄짤없이 3명
        -- RANK() OVER(ORDER BY 급여 DESC) AS rn        -- 공동 2등 있으면 3명
        DENSE_RANK() OVER(ORDER BY 급여 DESC) AS rn     -- 공동 순위 꽉꽉 채워서 3위까지 다 가져옴
    FROM 사원
)
SELECT 사원명, 급여, rn AS 순위
FROM RankedEmp
WHERE rn <= 3; -- 바깥 쿼리에서 3위까지만 싹둑 자른다!
```

#### 왜 굳이 `WITH` 절로 감싸야 할까? 

윈도우 함수는 SQL 실행 순서상 `WHERE` 절에서 바로 쓸 수가 없어. (`WHERE ROW_NUMBER() OVER(...) <= 3` ➔ **에러 발생!**)
무조건 안쪽 쿼리(`SELECT` 절)에서 순위표를 먼저 예쁘게 다 붙여놓고, 겉을 한 번 포장한 뒤에 바깥쪽 `WHERE` 절에서 필터링을 걸어줘야 해.

> 1. `FROM` → 2. `WHERE (ROWNUM 부여)` → 3. `ORDER BY`

---
## 초보자 실수

**"PostgreSQL에서 `LIMIT` 쓸 때 `ORDER BY`를 빼먹어도 대충 상위 데이터가 나오지 않나요?"**
- `LIMIT`이나 `ROWNUM`을 쓸 때는 **반드시 무엇을 기준으로 자를 것인지 `ORDER BY`를 명시**하는 게 철칙이야.

**"오라클에서 `WHERE ROWNUM = 3` 이라고 쓰면 딱 3등인 사람 한 명만 나오겠네요?"**
- **아무것도 안 나와! (데이터 0건)** `ROWNUM`은 항상 1번부터 순차적으로 번호표를 발급해야만 2번, 3번이 생기는 원리야. 1, 2번을 건너뛰고 갑자기 3번 번호표를 달라고 하면 조건을 영원히 만족하지 못해서 텅 빈 결과를 반환해. 특정 등수만 뽑고 싶다면 아까 배운 `ROW_NUMBER()` 윈도우 함수쓰기

>`FROM` → 2. `WHERE (ROWNUM 부여)` → 3. `ORDER BY`
>따라서 `ORDER BY`보다 `ROWNUM` 조건이 먼저 실행되면, 정렬도 안 된 '아무 데이터' 3개를 먼저 뽑고 그 안에서 정렬을 해버리는 대참사가 발생한다. 그래서 반드시 **인라인 뷰**를 써서 순서를 강제해야 한다.
