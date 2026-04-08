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
  - "[[SQL_Stored_Function]]"
---
# SQL Top-N 쿼리 — 상위 N개 데이터 추출하기

## 개념 한 줄 요약

> **"수십만 건의 데이터 중 정렬 기준에 따라 상위(또는 하위) N개 행만 쏙 뽑아내는 기법."**

---

## 왜 필요한가?

"가장 비싼 상품 3개", "최근 가입한 유저 5명" 을 찾을 때 테이블 전체를 다 불러올 필요가 없다. DB 서버 메모리·네트워크 부하를 줄이고, 게시판 페이징(1페이지당 10개씩) 처리에도 필수다.

> 실무(PostgreSQL) 에서는 `LIMIT` 으로 간단히 해결하지만, **SQLD 시험** 에서는 Oracle 의 `ROWNUM` 작동 원리를 집요하게 물어보기 때문에 둘 다 정확히 알아야 한다.

---

## DB별 Top-N 방법 한눈에 비교

| 방법                                         | DB                 | 특징                       |
| ------------------------------------------ | ------------------ | ------------------------ |
| `LIMIT`                                    | PostgreSQL · MySQL | 직관적. 쿼리 맨 끝에 붙이면 끝       |
| `ROWNUM`                                   | Oracle (구버전)       | 번호가 부여되는 시점 때문에 인라인 뷰 필수 |
| `FETCH FIRST N ROWS ONLY`                  | Oracle 12c+        | 표준 문법. LIMIT 처럼 사용 가능    |
| `ROW_NUMBER()` / `RANK()` / `DENSE_RANK()` | 모든 DB              | 파티션별 Top-N, 동점자 처리에 강력   |

---

---

# ① LIMIT — PostgreSQL · MySQL

가장 직관적인 방법. 쿼리 맨 마지막에 `LIMIT N` 만 붙이면 끝이다.

```sql
-- 급여가 가장 높은 3명 추출
SELECT 사원명, 급여
FROM 사원
ORDER BY 급여 DESC
LIMIT 3;
```

## OFFSET — 페이징 처리

```sql
-- 기본 구조
SELECT 컬럼 FROM 테이블
ORDER BY 기준컬럼
LIMIT 가져올개수 OFFSET 건너뛸개수;
```

```sql
-- 게시판 2페이지 (11~20번 게시글)
SELECT * FROM 게시글
ORDER BY 작성일 DESC
LIMIT 10 OFFSET 10;  -- 앞 10개 건너뛰고 다음 10개

-- 3등~5등 추출
SELECT 사원명, 급여
FROM 사원
ORDER BY 급여 DESC
LIMIT 3 OFFSET 2;   -- 앞 2개(1,2등) 건너뛰고 3개(3,4,5등) 가져옴
```

> **`ORDER BY` 는 필수다.** `LIMIT` 만 쓰고 `ORDER BY` 를 빼면 DB 가 임의의 순서로 N개를 반환한다. "대충 상위 데이터가 나오겠지" 는 착각이다.

---

---

# ② ROWNUM — Oracle (SQLD 핵심)

Oracle 에는 예전에 `LIMIT` 이 없었다. 대신 행이 출력되는 순서대로 1, 2, 3... 번호표를 붙여주는 **`ROWNUM`** 이라는 가상 컬럼을 사용했다.

## ROWNUM 의 절대 규칙

> **ROWNUM 은 1번부터 순차적으로만 발급된다. 건너뛰기 불가!**

은행 대기 번호표처럼 1번을 먼저 받아야 2번이 나오고, 2번을 받아야 3번이 나온다. 1, 2번을 건너뛰고 갑자기 3번 번호표를 달라고 하면 **영원히 조건을 만족하지 못해서 0건 반환**.

```sql
-- ✅ 가능: 1번부터 순차적으로 자르기
WHERE ROWNUM <= 3    -- 1, 2, 3번 → OK
WHERE ROWNUM < 4     -- 1, 2, 3번 → OK

-- ❌ 불가: 건너뛰기
WHERE ROWNUM = 3     -- 1, 2번을 건너뛰고 3번? → 0건 반환
WHERE ROWNUM >= 2    -- 1번을 건너뛰고 2번부터? → 0건 반환
WHERE ROWNUM > 1     -- 1번을 건너뛰어야 하므로 → 0건 반환
```

---

## SQL 실행 순서와 ROWNUM 의 함정

```
SQL 실행 순서:
① FROM    → 테이블 읽기
② WHERE   → ROWNUM 번호표 부여 + 필터링  ← ROWNUM 은 여기서 붙는다
③ ORDER BY → 정렬
```

`ROWNUM` 은 `ORDER BY` **이전** 에 붙는다. 즉, **정렬도 안 된 '아무 데이터' 에 먼저 번호표를 붙이고, 그다음에 정렬** 해버린다.

```sql
-- ❌ 대참사: 이렇게 짜면 절대 안 됨
SELECT 사원명, 급여
FROM 사원
WHERE ROWNUM <= 3     -- ① 정렬 전에 아무나 3명에게 번호표 붙임
ORDER BY 급여 DESC;   -- ② 그 3명 안에서만 정렬 → 최고 연봉자 3명이 아님!
```

---

## 올바른 ROWNUM 사용법 — 인라인 뷰 (서브쿼리)

정렬을 **먼저** 완료한 뒤, 바깥 쿼리에서 ROWNUM 으로 자른다.

```sql
-- ✅ 정답: 안쪽에서 정렬 완료 → 바깥에서 ROWNUM 으로 자르기
SELECT *
FROM (
    SELECT 사원명, 급여
    FROM 사원
    ORDER BY 급여 DESC   -- ① 인라인 뷰 안에서 정렬 완료
)
WHERE ROWNUM <= 3;        -- ② 바깥에서 위에서부터 3명만 자르기
```

```
실행 흐름:
인라인 뷰 안 → 전체 정렬 완료 (급여 내림차순)
바깥 WHERE  → 정렬된 결과에 ROWNUM 1, 2, 3 부여 후 자르기
```

---

## ROWNUM 으로 범위 추출 — 3등~5등

```sql
-- ✅ 3등~5등 추출 (ROWNUM 으로 건너뛰기는 불가 → ROW_NUMBER 로 처리)
SELECT *
FROM (
    SELECT 사원명, 급여,
           ROWNUM AS rn
    FROM (
        SELECT 사원명, 급여
        FROM 사원
        ORDER BY 급여 DESC  -- 가장 안쪽: 정렬
    )
    WHERE ROWNUM <= 5       -- 중간: 1~5등까지 자르기
)
WHERE rn >= 3;              -- 바깥: 3등 이상만 필터링 → 3,4,5등
```

> **왜 이렇게 3겹으로 감싸나?** `ROWNUM > 1` 은 절대 성립 안 된다고 했으므로, "1~5등 자르기" 를 먼저 한 뒤 "3등 이상 필터" 를 따로 걸어야 하기 때문이다.

---

## Oracle 12c — FETCH FIRST N ROWS ONLY

Oracle 12c 버전부터 표준 문법이 추가되어 `LIMIT` 처럼 사용할 수 있다.

```sql
-- LIMIT 처럼 사용
SELECT 사원명, 급여
FROM 사원
ORDER BY 급여 DESC
FETCH FIRST 3 ROWS ONLY;

-- OFFSET 도 가능
SELECT 사원명, 급여
FROM 사원
ORDER BY 급여 DESC
OFFSET 2 ROWS FETCH NEXT 3 ROWS ONLY;  -- 3등~5등
```

> SQLD 시험은 아직 ROWNUM 기준이 많으므로 두 가지 모두 알아두자.

---

---

# ③ 윈도우 함수 — ROW_NUMBER · RANK · DENSE_RANK

모든 DB 에서 통하는 실무 표준 방식. 특히 **"부서별 연봉 탑 3"** 처럼 파티션별 Top-N 이나 **동점자 처리** 를 디테일하게 결정할 때 독보적으로 강력하다.

## 세 함수 차이

|함수|동점자 처리|다음 순위|`<= 3` 으로 자르면|
|---|---|---|---|
|`ROW_NUMBER()`|동점 무시, 순서대로 번호|연속|**딱 3명**|
|`RANK()`|동점자에게 같은 순위|건너뜀 (1,2,2,4)|**3명** (4등 버려짐)|
|`DENSE_RANK()`|동점자에게 같은 순위|연속 (1,2,2,3)|**4명 이상 가능**|

```
예: 급여 500, 400, 400, 300 일 때

ROW_NUMBER : 1, 2, 3, 4    → <= 3 이면 500, 400, 400 (3명)
RANK        : 1, 2, 2, 4    → <= 3 이면 500, 400, 400 (3명, 4등 버려짐)
DENSE_RANK  : 1, 2, 2, 3    → <= 3 이면 500, 400, 400, 300 (4명!)
```

## 기본 사용법

```sql
-- 급여 순위를 붙이고, 바깥에서 3위까지만 필터링
WITH RankedEmp AS (
    SELECT
        사원명,
        급여,
        ROW_NUMBER() OVER (ORDER BY 급여 DESC) AS rn   -- 얄짤없이 정확히 N명
        -- RANK()       OVER (ORDER BY 급여 DESC) AS rn  -- 공동 순위 허용
        -- DENSE_RANK() OVER (ORDER BY 급여 DESC) AS rn  -- 공동 순위 + 연속 번호
    FROM 사원
)
SELECT 사원명, 급여, rn AS 순위
FROM RankedEmp
WHERE rn <= 3;
```

## 파티션별 Top-N — 부서별 연봉 1위

```sql
WITH RankedEmp AS (
    SELECT
        부서명,
        사원명,
        급여,
        ROW_NUMBER() OVER (
            PARTITION BY 부서명    -- 부서별로 따로 순위를 매김
            ORDER BY 급여 DESC
        ) AS rn
    FROM 사원
)
SELECT 부서명, 사원명, 급여
FROM RankedEmp
WHERE rn = 1;   -- 각 부서에서 1위만 추출
```

## 왜 WITH 절로 감싸야 하나?

윈도우 함수는 SQL 실행 순서상 `WHERE` 절에서 직접 쓸 수 없다.

```sql
-- ❌ 에러: WHERE 절에서 윈도우 함수 직접 사용 불가
SELECT 사원명, 급여
FROM 사원
WHERE ROW_NUMBER() OVER (ORDER BY 급여 DESC) <= 3;  -- 에러!

-- ✅ WITH 절(CTE) 또는 인라인 뷰로 감싸야 함
```

`SELECT` 절에서 먼저 순위 번호를 다 붙여놓고, 그 결과를 WITH 절로 포장한 뒤 바깥 `WHERE` 에서 필터링해야 한다.

> 윈도우 함수 상세 → [[SQL_Window_Functions]] 참고

---

---

# ROWNUM vs LIMIT vs ROW_NUMBER 최종 비교

|구분|`ROWNUM`|`LIMIT`|`ROW_NUMBER()`|
|---|---|---|---|
|DB|Oracle|PostgreSQL · MySQL|모든 DB|
|인라인 뷰 필요|✅ 필수|❌ 불필요|✅ WITH 절 필요|
|건너뛰기 (OFFSET)|⚠️ 3겹 서브쿼리|✅ OFFSET 키워드|✅ WHERE rn >= N|
|파티션별 Top-N|❌ 불가|❌ 불가|✅ PARTITION BY|
|동점자 처리|❌ 불가|❌ 불가|✅ RANK / DENSE_RANK|
|SQLD 출제 빈도|⭐️⭐️⭐️ 높음|낮음|⭐️⭐️ 중간|

---

