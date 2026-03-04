---
aliases:
  - SQL 피벗
  - SQL 언피벗
  - 행열변환
  - UNION ALL
  - FILTER
tags:
  - SQL
related:
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_UNION]]"
---
# SQL PIVOT · UNPIVOT — 행을 열로, 열을 행으로

## 개념 한 줄 요약

> **`PIVOT` : 세로로 긴 행(Row) → 가로로 넓은 열(Column) 로 펼치기**
> **`UNPIVOT` : 가로로 넓은 열(Column) → 세로로 긴 행(Row) 으로 해체하기**

---

## 왜 필요한가?

DB 에는 데이터가 발생 순서대로 **세로(Long format)** 로 쌓인다. 사람이 보고서를 읽을 때는 "가로축 = 월별, 세로축 = 부서별" 같은 **가로(Wide format)** 가 훨씬 직관적이다.

> **SQLD 시험:** Oracle 의 `PIVOT` · `UNPIVOT` 문법 구성요소(집계함수 / FOR / IN) 의 역할을 묻는 빈칸 문제 단골 출제. 
> **실무(PostgreSQL):** `PIVOT` 명령어 없음 → `FILTER` 절 또는 `UNION ALL` 로 수동 구현.

---

---

# ① PIVOT — 행을 열로 눕히기

## 원본 데이터 (Long format)

|부서명|월|매출|
|---|---|---|
|영업부|1월|100|
|영업부|2월|200|
|마케팅부|1월|300|
|마케팅부|2월|400|

## A. Oracle (SQLD) — PIVOT 연산자

```sql
SELECT * FROM 부서별매출
PIVOT (
    SUM(매출) AS SAL      -- ① 집계함수 + 별칭 (헤더명 맨 뒤에 붙음)
    FOR 월                -- ② FOR: 어떤 컬럼을 기준으로 쪼갤 건지
    IN ('1월' AS JAN,     -- ③ IN: 어떤 값들을 가로로 눕힐 건지 + 별칭 (헤더명 맨 앞에 붙음)
        '2월' AS FEB)
);
```

### 별칭(Alias) 합성 규칙 — SQLD 빈출 ⭐️

```
최종 헤더명 = IN절 별칭  +  _  +  집계함수 별칭
               ↑ 앞(접두사)       ↑ 뒤(접미사)

JAN  +  _  +  SAL  →  JAN_SAL
FEB  +  _  +  SAL  →  FEB_SAL
```

### PIVOT 결과 (Wide format)

|부서명|JAN_SAL|FEB_SAL|
|---|---|---|
|영업부|100|200|
|마케팅부|300|400|

---

## B. PostgreSQL (실무) — FILTER 절

PostgreSQL 에는 `PIVOT` 연산자가 없다. `FILTER` 로 수동 구현한다. 별칭 합성 규칙 같은 것도 없고, `AS` 뒤에 적은 이름이 그대로 헤더명이 된다.

```sql
SELECT
    부서명,
    SUM(매출) FILTER (WHERE 월 = '1월') AS "1월_매출",
    SUM(매출) FILTER (WHERE 월 = '2월') AS "2월_매출"
FROM 부서별매출
GROUP BY 부서명;  -- PostgreSQL 은 집계 기준 컬럼을 GROUP BY 에 반드시 명시
```

> FILTER 상세 → [[SQL_CASE_WHEN#PostgreSQL — FILTER 구문|PostegreSQL-FILTER]] 참고

---

---

# ② UNPIVOT — 열을 행으로 세우기

PIVOT 의 반대. 가로로 넓은 표를 다시 세로로 해체한다. **집계를 하는 게 아니라 해체만 하므로 집계함수가 없다.**

## 원본 데이터 (Wide format — PIVOT 결과물)

|부서명|1월_매출|2월_매출|
|---|---|---|
|영업부|100|NULL|
|마케팅부|300|400|

## A. Oracle (SQLD) — UNPIVOT 연산자

```sql
SELECT * FROM 피벗된_테이블
UNPIVOT INCLUDE NULLS (    -- ① NULL 행도 살리는 옵션 (기본은 EXCLUDE = 삭제)
    매출액                 -- ② 숫자들이 모일 새 컬럼명 (집계함수 아님!)
    FOR 해당월             -- ③ 기존 헤더명이 데이터로 담길 새 컬럼명
    IN (
        "1월_매출" AS '1월',  -- ④ 기존 컬럼명 → FOR 컬럼에 담길 실제 데이터 값
        "2월_매출" AS '2월'
    )
);
```

### 각 구성요소 역할 정리

|구성요소|역할|예시|
|---|---|---|
|`INCLUDE NULLS`|NULL 값 행도 결과에 포함 (기본은 삭제)|영업부 2월 NULL 행 살리기|
|`매출액`|흩어진 숫자들이 모일 새 컬럼명|100, NULL, 300, 400|
|`FOR 해당월`|기존 헤더명이 데이터로 내려올 새 컬럼명|'1월', '2월'|
|`IN (컬럼 AS '값')`|해체할 기존 컬럼 + FOR 컬럼에 들어갈 텍스트 값 변경|`"1월_매출"` → `'1월'`|

### UNPIVOT 결과

|부서명|해당월|매출액|
|---|---|---|
|영업부|1월|100|
|영업부|2월|NULL (`INCLUDE NULLS` 덕분에 살아남음)|
|마케팅부|1월|300|
|마케팅부|2월|400|

> **IN 절 AS 의 역할 차이:** 
> PIVOT IN 절 AS → **헤더명(컬럼명)** 을 바꾼다
> UNPIVOT IN 절 AS → **데이터 값(텍스트)** 을 바꾼다

---

## B. PostgreSQL (실무) — UNION ALL

```sql
SELECT 부서명, '1월' AS 해당월, "1월_매출" AS 매출액
FROM 피벗된_테이블

UNION ALL

SELECT 부서명, '2월' AS 해당월, "2월_매출" AS 매출액
FROM 피벗된_테이블;
```

> **UNION ALL 헤더명 규칙:** 최종 헤더명은 **첫 번째 쿼리의 별칭만** 따라간다. 두 번째 쿼리의 별칭은 무시되므로, 첫 번째 쿼리에 정확히 달아줘야 한다.

> **왜 UNION 이 아닌 UNION ALL?** 중복 제거 없이 100% 그대로 이어 붙이므로 속도가 빠르다. 
> 실무에서 특별한 이유가 없으면 항상 `UNION ALL` 이 정배. → [[SQL_UNION#1. `UNION ALL` 묻지도 따지지도 않고 다 붙이기 (가장 빠름)|UNION ALL]] 참고

---

---

# PIVOT vs UNPIVOT 최종 비교

|구분|PIVOT (행 → 열)|UNPIVOT (열 → 행)|
|---|---|---|
|목적|세로 데이터를 가로 요약표로|가로 데이터를 세로로 해체|
|집계함수|✅ **필수** (SUM, MAX 등)|❌ **없음** (해체만 함)|
|Oracle AS 역할|IN 별칭 → **헤더명(컬럼명)** 변경|IN 별칭 → **데이터 값** 변경|
|FOR 절 의미|쪼갤 기준이 될 **기존 컬럼**|헤더명이 데이터로 담길 **새 컬럼명**|
|IN 절 의미|가로로 눕힐 **실제 데이터 값**|세로로 해체할 **기존 열 이름**|
|NULL 처리|없으면 자동으로 NULL 채움|기본값 = NULL 행 삭제 (`EXCLUDE NULLS`)|
|NULL 살리기|(해당 없음)|`INCLUDE NULLS` 옵션|
|PostgreSQL 대체|`FILTER` + `GROUP BY`|`UNION ALL`|

---

## ⚠️ PIVOT → UNPIVOT 이 완벽한 복구가 아닌 이유

```
원본: 영업부 1월 매출 50원, 50원 (2건)
         ↓ PIVOT (SUM)
피벗: 영업부 | 1월_매출 = 100원 (압축됨)
         ↓ UNPIVOT
결과: 영업부 | 1월 | 100원 (1건)  ← 50원짜리 2건으로 복구 불가!
```

> PIVOT 과정에서 `SUM` 으로 데이터가 **압축(손실)** 되었기 때문에 UNPIVOT 으로 되돌려도 원본으로 완벽히 복구되지 않는다.

---

