---
aliases:
  - SQL NULL
  - NULL 연산
  - IS NULL
  - NVL
  - COALESCE
tags:
  - SQL
related:
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[SQL_NULL_Functions]]"
  - "[[SQL_SELECT_FROM]]"
---


# SQL NULL 이해 (Understanding NULL)

## 개념 한 줄 요약

> **"공백도 아니고 0도 아닌, '아직 정의되지 않은 값(Unknown).'** 블랙홀처럼 무엇과 연산하든 결과는 무조건 NULL 이 되어버린다."

---

---

# ① 가로 연산 — NULL 이 전파된다

> 한 행(Row) 안에서 컬럼끼리 연산할 때의 규칙. **NULL 이 하나라도 섞이면 결과는 무조건 NULL.**

```sql
SELECT
    100 + 200  AS result_1,   -- 300
    100 + NULL AS result_2,   -- NULL  ← 100 이 사라짐!
    100 * NULL AS result_3    -- NULL
FROM dual;
```

## 실무 위험 상황

```
급여 + 커미션 계산 시 커미션이 NULL 인 직원
→ 급여까지 NULL 이 되어 월급이 0원으로 찍히는 대참사
```

```sql
-- ❌ 위험: 커미션이 NULL 이면 총연봉도 NULL
SELECT 급여 + 커미션 AS 총연봉 FROM 사원;

-- ✅ 안전: NULL → 0 으로 변환 후 연산
SELECT 급여 + NVL(커미션, 0)      AS 총연봉 FROM 사원;  -- Oracle
SELECT 급여 + COALESCE(커미션, 0) AS 총연봉 FROM 사원;  -- 표준
```

> → [[SQL_NULL_Functions]] — NVL · COALESCE · NULLIF · NVL2

---

---

# ② 세로 연산 — NULL 이 무시된다

> 집계 함수(`SUM`, `AVG` 등) 는 NULL 을 없는 셈 치고 계산한다. **가로 연산과 반대 방향이라 헷갈리기 쉽다.**

```sql
-- 데이터: [100, 200, NULL] (총 3건)
SELECT
    SUM(score) AS 합계,   -- 300  (NULL 무시)
    AVG(score) AS 평균    -- 150  (300 ÷ 2명)  ← 3명이 아님!
FROM exams;
```

## SQLD 함정 — 평균의 배신 ⭐️

|계산 방식|분모|결과|
|---|:-:|:-:|
|`AVG(score)`|2명 (NULL 제외)|**150**|
|`AVG(COALESCE(score, 0))`|3명 (NULL → 0)|**100**|

```sql
-- NULL 을 0점으로 처리해서 3명 기준으로 평균 내고 싶다면
SELECT AVG(COALESCE(score, 0)) FROM exams;  -- 결과: 100
```


## ⭐️ 예외 — 결과 자체가 NULL 이 되는 경우

> **더할 대상이 아예 없거나, 대상이 모두 NULL 뿐일 때는 0 이 아니라 NULL 이 반환된다.**

```sql
-- 케이스 ①: 조건에 맞는 행 자체가 없을 때
SELECT SUM(score) FROM exams WHERE class = '없는반';
-- 결과: NULL  (0 이 아님!)

-- 케이스 ②: 대상 컬럼이 전부 NULL 일 때
-- 데이터: [NULL, NULL, NULL]
SELECT SUM(score) FROM exams;   -- 결과: NULL
SELECT AVG(score) FROM exams;   -- 결과: NULL
```

> 더할 대상이 없거나 전부 NULL 일 때 결과가 0 이 아닌 NULL 로 반환되는 예외 케이스 → [[SQL_Aggregate_GROUP_BY#⭐️ 예외 — 결과 자체가 NULL 이 되는 경우|예외-결과자체가 NULL이 되는 경우 ]] 참고

---

---

# ③ 비교 연산 — 항상 Unknown

> NULL 을 `{text}=` 로 비교하면 항상 Unknown → 결과 없음. **반드시 `IS NULL` / `IS NOT NULL` 을 써야 한다.**

```sql
-- ❌ 아무 결과도 안 나옴
SELECT * FROM employees WHERE comm = NULL;

-- ✅ 정상
SELECT * FROM employees WHERE comm IS NULL;
SELECT * FROM employees WHERE comm IS NOT NULL;
```

## NULL = NULL 은 참일까?

```
아니다.
"알 수 없는 값" = "알 수 없는 값" → 같은지조차 알 수 없음 → Unknown

→ JOIN 시 양쪽 컬럼이 모두 NULL 이면 매칭되지 않는다 (조인 실패)
```

---

---

# ④ WHERE 절의 배신 — NULL 은 자동으로 증발한다

> 조건을 걸면 NULL 데이터는 아무 말 없이 사라진다. `NULL > 5` = Unknown → True 가 아니므로 행 전체 탈락.

## 예시 테이블 (SAMPLE)

|col1|col2|WHERE col2 > 5|
|---|---|:-:|
|10|10|✅ 통과 (10 > 5)|
|20|3|❌ 탈락 (3 ≤ 5)|
|30|NULL|❌ 탈락 (NULL > 5 = Unknown)|

```sql
SELECT SUM(col1), SUM(col2)
FROM SAMPLE
WHERE col2 > 5;
-- 결과: SUM(col1)=10, SUM(col2)=10
-- 3번째 행은 col1=30 이지만 col2=NULL 때문에 행 전체 버려짐!
```

## 해결책

```sql
WHERE NVL(col2, 0) > 5          -- NULL 을 0 으로 바꿔서 비교
WHERE col2 > 5 OR col2 IS NULL  -- NULL 도 명시적으로 포함
```

---

---

# ⑤ IN / NOT IN + NULL 함정 ⭐️ SQLD 단골

## IN + NULL — 착각 금지 ⚠️

> **`IN(1, 2, NULL)` 처럼 NULL 을 리스트에 넣어도 NULL 행은 절대 찾지 못한다.** SQLD 에서 "NULL 을 넣으면 NULL 행도 같이 찾아준다" 고 착각하기 쉬운 대표 함정.

```sql
-- 데이터: col1 = [1, 2, NULL, 3]

SELECT * FROM T WHERE col1 IN (1, 2, NULL);
```

```
col1 = 1    →  1 IN (1, 2, NULL)    = TRUE    → 출력 ✅
col1 = 2    →  2 IN (1, 2, NULL)    = TRUE    → 출력 ✅
col1 = NULL →  NULL IN (1, 2, NULL) = UNKNOWN → 탈락 ❌  ← 핵심!
col1 = 3    →  3 IN (1, 2, NULL)    = FALSE   → 탈락 ❌
```

> **NULL 행을 찾고 싶으면 IS NULL 을 별도로 써야 한다.**

```sql
-- NULL 행까지 포함하고 싶을 때
WHERE col1 IN (1, 2) OR col1 IS NULL
```

---

## NOT IN + NULL — 결과가 0건 ⚠️

> **리스트 안에 NULL 이 하나라도 있으면 전체 결과가 0건.**

```sql
SELECT * FROM T WHERE col1 NOT IN (10, 20, NULL);
-- 결과: 0건 (아무것도 안 나옴)
```

```
NOT IN = != AND != AND != 연산으로 풀림

col1 != 10 AND col1 != 20 AND col1 != NULL
                                    ↑
                              != NULL = UNKNOWN
→ 전체 조건이 UNKNOWN → 모든 행 탈락
```

```sql
-- ✅ 안전하게 쓰기: 서브쿼리에서 반드시 NULL 제거
WHERE col1 NOT IN (
    SELECT col1 FROM T WHERE col1 IS NOT NULL
)
```

---

## IN vs NOT IN + NULL 한눈에 비교

|상황|결과|이유|
|---|---|---|
|`IN (1, 2, NULL)`|1, 2 에 해당하는 행만 출력|NULL 행은 UNKNOWN 으로 탈락|
|`NOT IN (1, 2, NULL)`|**0건**|`!= NULL` = UNKNOWN → 전체 탈락|

```
IN  + NULL →  다른 값은 정상 동작, NULL 행만 못 찾음
NOT IN + NULL →  리스트에 NULL 하나라도 있으면 전체 0건
```

---

---

# ⑥ 정렬 시 NULL 위치 (ORDER BY)

|DB|NULL 취급|오름차순 위치|
|---|---|:-:|
|**Oracle** ⭐️ SQLD 기준|가장 큰 값|맨 뒤|
|SQL Server|가장 작은 값|맨 앞|

```sql
ORDER BY salary DESC NULLS FIRST;   -- NULL 을 맨 앞으로
ORDER BY salary ASC  NULLS LAST;    -- NULL 을 맨 뒤로
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|`WHERE col = NULL`|`=` 로 NULL 비교 불가|`IS NULL` 사용|
|AVG 결과가 예상과 다름|NULL 제외하고 계산|`COALESCE(col, 0)` 으로 대체|
|연산 결과가 NULL|가로 연산 시 NULL 전파|`NVL` / `COALESCE` 로 0 변환 후 연산|
|`IN(1,2,NULL)` 으로 NULL 행을 찾으려 함|NULL 비교는 항상 UNKNOWN|`OR col IS NULL` 별도 추가|
|`NOT IN(...)` 결과 0건|리스트 안 NULL 때문에 전체 UNKNOWN|서브쿼리에 `IS NOT NULL` 조건 추가|
|`COUNT(컬럼)` 이 전체 행 수보다 적음|NULL 행 제외|전체 행 수는 `COUNT(*)` 사용|

---

