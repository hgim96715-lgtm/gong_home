---
aliases:
  - CASE WHEN
  - SQL IF문
  - 조건문
  - 조건부 집계
  - FILTER
tags:
  - SQL
related:
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_Pivot_Unpivot]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
---
# SQL CASE WHEN — 쿼리 속의 IF-ELSE 분기점

## 개념 한 줄 요약

> **"만약(IF) ~라면 A, 아니면(ELSE) B" 라는 논리를 SQL 에서 구현하는 문법.** 엑셀의 `IF` 함수, 프로그래밍의 `if-else` 문과 똑같다.

---

## 언제 쓰나?

|사용 목적|예시|
|---|---|
|데이터 범주화|점수(85점) → 등급('B학점') 으로 변환|
|데이터 정제|`'M'` · `'F'` → `'Male'` · `'Female'` 로 변환|
|피벗 테이블|세로 데이터를 월별 컬럼(1월, 2월, 3월)으로 가로로 펼치기|
|조건부 집계|전체 매출 중 '전자제품' 매출만 별도 컬럼으로 추출|
|비율 계산|전체 건수 대비 특정 조건 건수 비율 구하기|

---

---

# ① 기본 문법

## SEARCHED CASE — 조건식 전체를 쓰는 일반적인 형태

```sql
SELECT
    student_name,
    score,
    CASE
        WHEN score >= 90 THEN 'A학점'
        WHEN score >= 80 THEN 'B학점'
        WHEN score >= 70 THEN 'C학점'
        ELSE 'F학점'          -- 나머지 전부. ELSE 생략 시 → NULL 반환
    END AS grade              -- END 로 반드시 닫아야 한다
FROM exams;
```

## SIMPLE CASE — 비교할 컬럼을 CASE 바로 뒤에 선언

```sql
SELECT LOC,
    CASE LOC
        WHEN 'NEW YORK' THEN 'EAST'
        WHEN 'CHICAGO'  THEN 'MIDWEST'
        ELSE 'ETC'
    END AS AREA
FROM DEPT;
```

## 두 형태 비교

|구분|구조|WHEN 뒤에 오는 것|범위 비교 가능?|
|---|---|---|---|
|SEARCHED CASE|`CASE WHEN 조건식 THEN`|`score >= 90` 같은 **조건식 전체**|✅ 가능|
|SIMPLE CASE|`CASE 컬럼 WHEN 값 THEN`|`'NEW YORK'` 같은 **값만**|❌ `=` 비교만 가능|

> **SIMPLE CASE 치명적 한계:** 정확히 일치(`{text}=`) 비교만 가능하다. 대소 비교(`>`, `<`, `>=`) 가 필요하면 반드시 **SEARCHED CASE** 를 써야 한다.

```sql
-- ❌ SIMPLE CASE 로 범위 비교 불가
CASE score WHEN >= 90 THEN 'A'  -- 문법 에러!
 
-- ✅ 범위 비교는 SEARCHED CASE 만 가능
CASE WHEN score >= 90 THEN 'A'
```

---

## ELSE 와 NULL 의 관계

|상황|결과|
|---|---|
|`ELSE 값` 명시|조건에 안 맞는 행은 그 값으로 채워짐|
|`ELSE` 생략|조건에 안 맞는 행은 **자동으로 NULL**|
|`ELSE ''` 명시|NULL 이 아닌 **빈 문자열(공백)** 이 들어감|

> `ELSE` 를 생략하면 데이터 빵꾸(NULL)의 주범이 된다. 항상 명시하는 습관을 들이자.

---

---

# ② DB 별 전용 문법

## Oracle — DECODE 함수

`CASE WHEN` 의 조상 격인 Oracle 전용 함수다.

### 기본 문법

```
DECODE(컬럼, 조건1, 결과1, 조건2, 결과2, ..., 기본값)
```

```sql
-- 성별 코드 변환 (CASE WHEN 과 동일한 결과)
SELECT
    student_name,
    DECODE(gender, 'M', 'Male', 'F', 'Female', 'Unknown') AS gender_name
FROM students;
```

### DECODE 인수 개수와 DEFAULT 값 ⭐️

> **인수 개수가 홀수면 마지막 값이 DEFAULT 가 된다.** 짝수면 DEFAULT 없이 조건쌍만 있는 것이므로, 조건에 맞지 않으면 **NULL 반환**.

```sql
-- ✅ 홀수 (마지막 'Unknown' 이 DEFAULT)
DECODE(gender, 'M', 'Male', 'F', 'Female', 'Unknown')
--              ↑     ↑      ↑     ↑         ↑
--              조건1  결과1  조건2  결과2    DEFAULT

-- ⚠️ 짝수 (DEFAULT 없음 → 매칭 안 되면 NULL 반환)
DECODE(gender, 'M', 'Male', 'F', 'Female')
-- gender 가 'M' 도 'F' 도 아니면 → NULL

-- ✅ 짝수이지만 마지막에 '' (빈 문자열) 로 DEFAULT 지정
DECODE(gender, 'M', 'Male', 'F', 'Female', '')
-- gender 가 'M' 도 'F' 도 아니면 → NULL 이 아닌 빈 문자열(공백) 출력
--                                          ↑
--                                      '' 는 NULL 이 아니라 빈 문자열이다!
```

> **치명적 단점:** `CASE WHEN` 처럼 대소 비교(`>`, `<`) 는 불가능. 오직 **정확히 일치(`{text}=`)** 할 때만 사용할 수 있다.

---

## PostgreSQL — FILTER 구문

조건부 집계를 위해 태어난 최신 문법. 코드가 훨씬 직관적이다.

### 문법 구조

```text
집계함수(*) FILTER (WHERE 조건)
    ↑                    ↑
  평소처럼 집계       이 조건일 때만 집계
```

### COUNT + FILTER

```sql
SELECT
    COUNT(*) FILTER (WHERE credit ILIKE '%gift%')     AS gift_count,
    COUNT(*) FILTER (WHERE credit NOT ILIKE '%gift%') AS non_gift_count,
    COUNT(*)                                           AS total_count
FROM payments;
-- "COUNT 는 하되, 이 조건일 때만 세어줘"
```

```text
ILIKE     → 대소문자 무시하고 패턴 매칭  ('Gift', 'GIFT', 'gift' 모두 잡음)
NOT ILIKE → 패턴에 해당하지 않는 것만   ('gift' 가 포함되지 않은 행만)
```

### SUM + FILTER

```sql
SELECT
    SUM(amount) FILTER (WHERE credit ILIKE '%gift%')     AS gift_총액,
    SUM(amount) FILTER (WHERE credit NOT ILIKE '%gift%') AS 일반_총액,
    SUM(amount)                                           AS 전체_총액
FROM payments;
-- "금액을 더하되, 기프트카드 결제만 / 일반 결제만 / 전체 따로따로 집계"
```

### FILTER vs CASE WHEN 비교

```sql
-- ✅ FILTER (PostgreSQL 전용, 직관적)
COUNT(*) FILTER (WHERE credit ILIKE '%gift%')

-- ✅ CASE WHEN (표준 SQL, 모든 DB 호환)
SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END)
```

|방식|가독성|DB 호환|
|---|---|---|
|`FILTER`|✅ 직관적|PostgreSQL 전용|
|`CASE WHEN`|조금 길다|모든 DB ✅|

> **SQLD 시험은 Oracle 기준 → CASE WHEN 으로 써야 한다.**
>  **PostgreSQL 실무 → FILTER 가 훨씬 간결하다.**

---

## BigQuery — COUNTIF 함수

```sql
SELECT
    COUNTIF(credit LIKE '%gift%') AS gift_count,
    COUNTIF(credit LIKE '%gift%') / COUNT(*) AS gift_ratio
FROM payments;
```

---

---

# ③ 조건부 집계 — CASE WHEN 의 진짜 파괴력

> **초보자는 필터링할 때 `WHERE` 를 쓰고, 고수는 `CASE WHEN` 을 쓴다.**

## A. 조건부 카운트 — 1과 0 더하기

특정 조건에 맞으면 1, 아니면 0을 부여하고 `SUM` 해버리는 테크닉이다.

```sql
SELECT
    product_name,
    SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END) AS sales_2023,
    SUM(CASE WHEN year = 2024 THEN amount ELSE 0 END) AS sales_2024,
    -- 두 컬럼이 나란히 생성되므로 즉시 증감 계산 가능!
    SUM(CASE WHEN year = 2024 THEN amount ELSE 0 END) -
    SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END) AS diff
FROM yearly_sales
GROUP BY product_name;
```

**GROUP BY 방식 vs CASE WHEN 피벗 비교:**

|방식|결과 형태|뺄셈 계산|
|---|---|---|
|`GROUP BY year`|세로로 2줄 (2023행, 2024행)|❌ 불가능|
|`CASE WHEN` (피벗)|가로로 1줄 (2023컬럼, 2024컬럼)|✅ 바로 가능|

---

## B. 조건부 평균 — ELSE NULL 주의 ⚠️

```sql
-- ✅ PostgreSQL: FILTER 사용
SELECT
    ROUND(AVG(critic_score) FILTER (WHERE year = 2011), 2) AS score_2011
FROM game_analytics;

-- ✅ 표준 SQL: CASE WHEN + ELSE NULL
SELECT
    ROUND(AVG(CASE WHEN year = 2011 THEN critic_score ELSE NULL END), 2) AS score_2011
FROM game_analytics;

-- ❌ 절대 금지: AVG 에서 ELSE 0 사용
SELECT
    AVG(CASE WHEN year = 2011 THEN critic_score ELSE 0 END) AS score_2011
FROM game_analytics;
-- 조건 안 맞는 데이터가 0점으로 평균 계산(분모)에 포함 → 평균이 대폭락!
```

> **SUM 에서는 `ELSE 0`, AVG 에서는 `ELSE NULL` (또는 ELSE 생략)** 
> AVG 는 NULL 은 계산에서 제외하지만 0 은 포함해서 나누기 때문이다.

---

---

# ④ 비율 계산 — WHERE 를 쓰면 안 되는 이유 ⭐️

> **"비율(Ratio) 을 구할 땐 절대로 WHERE 를 쓰지 마라."**

## 왜 WHERE 가 틀리는가?

```sql
-- ❌ 잘못된 쿼리: WHERE 로 필터링
SELECT
    COUNT(*) AS gift_count,   -- 기프트카드 건수
    COUNT(*) AS total_count   -- 전체 건수라고 착각!
FROM payments
WHERE credit ILIKE '%gift%';  -- ⚠️ 여기서 기프트카드 아닌 행이 전부 날아감

-- WHERE 가 먼저 실행 → 기프트카드만 남음
-- → 분자(gift) = 분모(total) → 비율이 항상 100% !
```

## 올바른 비율 계산 — CASE WHEN 으로 분모 살리기

```sql
SELECT
    -- 분자: 조건에 맞는 것만 1로 바꿔서 합산
    SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END) AS gift_count,

    -- 분모: 전체 데이터 (아무것도 버리지 않음)
    COUNT(*) AS total_count,

    -- 비율 계산
    (SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END)::float
     / COUNT(*)) * 100 AS ratio
FROM payments;
```

## 클릭률(CTR) 계산 예제

```sql
SELECT
    ad_name,
    SUM(CASE WHEN action = 'click' THEN 1 ELSE 0 END)          AS click_count,
    COUNT(*)                                                     AS total_view,
    SUM(CASE WHEN action = 'click' THEN 1 ELSE 0 END) * 1.0
        / COUNT(*)                                               AS ctr
FROM ad_logs
GROUP BY ad_name;
-- * 1.0 을 곱하는 이유: 정수 / 정수 = 0 이 되는 걸 막기 위해 소수점 강제 생성
```

|방식|동작|
|---|---|
|`WHERE`|데이터를 **잘라내고 버림** (분모가 사라짐)|
|`CASE WHEN`|데이터를 **버리지 않고 표시만 함** (분모 유지)|

---
# ⑤ HAVING 안에서 CASE WHEN ⭐️

>**CASE WHEN 은 HAVING 안에서도 쓸 수 있다.** 
>특히 `COUNT(DISTINCT CASE WHEN ... END)` 패턴은 "특정 조건을 N개 이상 만족하는 그룹만 걸러낼 때" 강력하다.

## 패턴 공식

```sql
GROUP BY 기준컬럼
HAVING COUNT(DISTINCT CASE WHEN 조건 THEN 그룹값 END) >= N
```

```text
동작 원리:
① CASE WHEN 으로 조건에 맞는 행에만 값(그룹값)을 부여
② 조건에 안 맞으면 ELSE 없으니 NULL → COUNT 에서 자동 제외
③ DISTINCT 로 중복 제거
④ 그 결과가 N 개 이상인 그룹만 남김
```

## 실전 예시 — 2개 이상 플랫폼 제조사가 있는 게임만 조회

```sql
SELECT
    game_name
FROM platforms p
GROUP BY game_name
HAVING COUNT(DISTINCT
    CASE
        WHEN p.name IN ('PS3','PS4','PSP','PSV') THEN 'Sony'
        WHEN p.name IN ('Wii','WiiU','DS','3DS') THEN 'Nintendo'
        WHEN p.name IN ('X360','XONE')           THEN 'Microsoft'
    END
) >= 2;
```

## 언제 이 패턴을 쓰는가

```text
"최소 N개 이상의 카테고리에 속하는 그룹만 추출"

예시:
- 3개 이상 지역에서 판매된 상품
- 2개 이상 장르를 보유한 감독
- 2개 이상 제조사 플랫폼에 출시된 게임
```

>⚠️ **CTE 없이도 해결 가능하다.** 
>HAVING 안에 CASE WHEN 을 바로 넣으면 되기 때문에 서브쿼리나 CTE 로 먼저 분류한 뒤 다시 집계할 필요가 없다.



---

# 초보자가 자주 하는 실수

## ① ELSE 를 안 쓰면?

조건에 안 맞는 데이터는 자동으로 **`NULL`** 이 된다. 빈 값이 싫다면 `ELSE 'Unknown'` 처럼 기본값을 반드시 넣자.

## ② END 를 까먹으면?

`CASE` 를 열었으면 무조건 `END` 로 닫아야 한다. 에러 메시지에 `syntax error at or near...` 가 뜨면 90% 는 `END` 누락이다.

## ③ 비율 계산했는데 0이 나오면?

```sql
-- ❌ 정수 / 정수 = 정수 (소수점 버림)
2 / 5 = 0

-- ✅ 해결 방법
2 * 1.0 / 5   -- * 1.0 으로 실수 변환 (PostgreSQL · Oracle)
2::float / 5  -- ::float 로 형변환 (PostgreSQL)
```

## ④ AVG 에서 ELSE 0 을 쓰면?

조건에 안 맞는 값이 0으로 평균 계산에 포함되어 **평균이 대폭락** 한다. AVG 에서는 반드시 `ELSE NULL` 또는 ELSE 를 생략해야 한다.

---
