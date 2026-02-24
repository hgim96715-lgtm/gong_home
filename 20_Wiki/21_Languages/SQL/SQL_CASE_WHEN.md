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
  - "[[Pivot_Unpivot]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
---
# 쿼리 속의 IF-ELSE 분기점

## 개념 한 줄 요약

SQL에서 **"만약(IF) ~라면 A, 아니면(ELSE) B"** 라는 논리를 구현하는 문법이야.
엑셀의 `IF` 함수나 프로그래밍의 `if-else` 문과 똑같아.

---
## 왜 필요한가 (Why)

1.  **데이터 범주화 (Categorization):** 점수(85점)를 등급('A학점')으로 바꿀 때.
2.  **데이터 정제 (Cleaning):** `M`, `F`로 된 성별을 `Male`, `Female`로 바꿀 때.
3. **피벗 테이블(Pivot):** "월별 매출 추이"를 한 눈에 보고 싶을 때 (1월, 2월, 3월 컬럼 생성).
4.  **조건부 집계 (Conditional Aggregation):** (핵심🔥) 전체 매출 중 '전자제품' 매출만 따로 컬럼으로 뽑고 싶을 때.

---
##  Basic Syntax (기본 문법)

```sql
SELECT
    student_name,
    score,
    -- [문법 시작]
    CASE 
        WHEN score >= 90 THEN 'A학점'   -- 90점 이상이면 A
        WHEN score >= 80 THEN 'B학점'   -- 80점 이상이면 B
        ELSE 'C학점'                    -- 나머지 전부 [핵심] ELSE 뒤의 값이 '기본값'이 됨!
    END AS grade                        -- [문법 끝] 컬럼 별명(AS) 필수!
FROM exams;
```

**ELSE와 NULL의 관계**

- `CASE` 문에서 `ELSE` 뒤에 지정한 값이 해당 컬럼의 **기본값(Default)** 이 됩니다.
- 만약 쿼리에서 별도의 `ELSE` 구문을 작성하지 않으면, 조건에 맞지 않는 모든 데이터는 **무조건 `NULL` 값이 기본값**으로 반환됩니다. (데이터 빵꾸의 주범이 되므로 주의!)

---
## DB 별 전용 문법

표준 SQL인 `CASE WHEN`은 조금 길기 때문에, 각 DB마다 이를 압축한 전용 함수나 구문을 지원합니다.

### Oracle 전용: `DECODE` 함수 

Oracle에는 `CASE WHEN`의 조상 격인 `DECODE` 함수가 있습니다.

- **문법:** `DECODE(컬럼, 조건1, 결과1, 조건2, 결과2, ..., 기본값)`
- **치명적 단점:** `CASE WHEN`처럼 대소 비교(`>`, `<`)는 불가능하고, 오직 **정확히 일치(`{text}=`)** 할 때만 사용할 수 있습니다.

```sql
-- 성별 코드를 변환하는 예시 (CASE WHEN과 완벽히 동일한 결과)
SELECT
    student_name,
    DECODE(gender, 'M', 'Male', 'F', 'Female', 'Unknown') AS gender_name
FROM students;
```

### PostgreSQL 전용: `FILTER` 구문

"조건부 집계"를 위해 태어난 최신 문법입니다. 코드가 훨씬 직관적입니다.

```sql
SELECT
    -- "COUNT를 하긴 할 건데(COUNT*), 이 조건일 때만(FILTER) 세어줘"
    COUNT(*) FILTER (WHERE credit ILIKE '%gift%') AS gift_count,
    COUNT(*) AS total_count
FROM payments;
```

### BigQuery 전용: `COUNTIF` 함수

구글이 만든 초단축 함수입니다. "조건(IF)에 맞는 것만 세라(COUNT)"는 뜻입니다.

```sql
SELECT
    COUNTIF(credit LIKE '%gift%') AS gift_count,
    COUNTIF(credit LIKE '%gift%') / COUNT(*) AS gift_ratio -- 비율 계산이 압도적으로 깔끔함
FROM payments;
```

---
## 조건부 집계 (Conditional Aggregation) 

초보자는 `WHERE`를 쓰고, 고수는 `CASE WHEN`을 씁니다. 
특히 **"전체 중에 특정 조건의 비율(%)"** 을 구할 때 이 기술이 필수입니다.

### ❌ 초보자의 고민 (WHERE의 한계)

> "남자의 숫자도 세고 싶고, 여자의 숫자도 세고 싶은데... `WHERE gender = 'M'`을 하면 여자가 사라지고, `WHERE gender = 'F'`를 하면 남자가 사라져요!


### ⭕️ 해결책: `SUM` + `CASE WHEN` (피벗)

데이터를 버리지(`WHERE`) 말고, **살려둔 채로 표시만 다르게** 하는 겁니다.

- **원리:** 조건에 맞으면 **1**, 아니면 **0**을 부여하고 다 더해버림(`SUM`).

```sql
SELECT
    product_name,
    
    -- [2023년 컬럼] 2023년 데이터만 골라서 합계
    SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END) AS sales_2023,
    
    -- [2024년 컬럼] 2024년 데이터만 골라서 합계
    SUM(CASE WHEN year = 2024 THEN amount ELSE 0 END) AS sales_2024,
    
    -- [증감율 계산 가능] 이제 컬럼이 옆에 있으니까 바로 계산 가능!
    (SUM(CASE WHEN year = 2024 THEN amount ELSE 0 END) - 
     SUM(CASE WHEN year = 2023 THEN amount ELSE 0 END)) AS diff
     
FROM yearly_sales
GROUP BY product_name; -- 연도(year)로는 묶지 않음!
```

**결과 비교:**

|**방식**|**결과 형태**|**비고**|
|---|---|---|
|**GROUP BY year**|세로로 2줄 나옴 (2023, 2024)|뺄셈 계산 불가능|
|**CASE WHEN (Pivot)**|**가로로 1줄 나옴 (2023컬럼, 2024컬럼)**|**바로 뺄셈 가능**|

----
## 클릭률(CTR)과 비율 계산하기 

**"비율 계산은 WHERE 대신 조건부 집계로 처리한다"** 는 말이 바로 이 뜻입니다.

**Q. 광고를 본 사람 대비 클릭한 사람의 비율(CTR)은?**

- **분모:** 전체 노출 수 (`COUNT(*)`)
- **분자:** 클릭한 수 (`status = 'click'`)

```sql
SELECT
    ad_name,
    
    -- 1. 클릭 수 구하기 (분자)
    SUM(CASE WHEN action = 'click' THEN 1 ELSE 0 END) AS click_count,
    
    -- 2. 전체 노출 수 구하기 (분모)
    COUNT(*) AS total_view,
    
    -- 3. 클릭률(%) 계산 (분자 / 분모)
    -- 1.0을 곱하는 이유는 소수점을 만들기 위해서임 (안 하면 0으로 나옴)
    SUM(CASE WHEN action = 'click' THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS ctr
    
FROM ad_logs
GROUP BY ad_name;
```

---
## 🚫 치명적인 실수: WHERE로 필터링하면 '분모'가 사라진다!

많은 분들이 **"기프트카드(gift) 결제 비율을 구해라"** 라는 질문을 받으면 무의식적으로 `WHERE`를 씁니다. 하지만 이건 **오답**입니다.

### ❌ 잘못된 쿼리 (WHERE 사용)

```sql
SELECT 
    COUNT(*) AS gift_count,   -- 기프트카드 건수
    COUNT(*) AS total_count   -- 전체 건수 (라고 착각함)
FROM payments
WHERE credit ILIKE '%gift%'; -- ⚠️ 여기서 이미 전체 데이터가 날아감!
```

**결과:** `WHERE`가 먼저 실행되어 '기프트카드'가 아닌 행을 다 지워버렸습니다.
남은 건 기프트카드 뿐이라, **분자(gift)와 분모(total)가 같아져서 비율이 항상 100%** 가 나옵니다.


### ⭕️ 올바른 쿼리 (CASE WHEN 사용)

전체 데이터(분모)를 **살려둔 채로**, 원하는 것만 **콕 집어서(CASE WHEN)** 세야 합니다.

```sql
SELECT 
    -- 1. 분자: 조건에 맞는 것만 1로 바꿔서 셈 (ILIKE 사용 가능)
    SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END) AS gift_count,

    -- 2. 분모: 아무 조건 없이 다 셈 (전체 데이터 생존)
    COUNT(*) AS total_count,

    -- 3. 비율 계산 가능
    (SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END)::float / COUNT(*)) * 100 AS ratio
FROM payments;
```

**WHERE**: 데이터를 **잘라내고 버림**. (남은 것끼리만 계산)
**CASE WHEN**: 데이터를 **버리지 않고 표시만 함**. (전체 대비 비율 계산 가능)
**공식:** **"비율(Ratio)을 구할 땐 절대로 WHERE를 쓰지 마라."**

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "ELSE를 안 쓰면 어떻게 되나요?"

- 조건에 안 맞는 데이터는 자동으로 **`NULL`** 이 됩니다.
- `CASE WHEN score > 90 THEN 'A'` 만 쓰면, 80점은 `NULL`이 됩니다. 빈 값이 싫다면 `ELSE 'Unknown'` 처럼 기본값을 꼭 넣어주세요.

### ② "END를 자꾸 까먹어요."

- SQL에서 `CASE`를 열었으면 무조건 **`END`** 로 닫아줘야 합니다. 괄호를 닫는 것(`)`)과 같은 이치입니다.
- 에러 메시지에 `syntax error at or near...`가 뜨면 90%는 `END` 누락입니다.

### ③ "비율 계산했는데 0이 나와요!"

- SQL(특히 Postgres)에서 `정수 / 정수 = 정수`입니다. (예: `2 / 5 = 0`)
- 분자나 분모 중 하나에 **`* 1.0`** 을 곱하거나 `::float`로 형변환을 해야 소수점이 나옵니다.

---
## 요약

1. 단순 변환: **"점수 ➔ 등급"** 바꿀 때 쓴다.
2. 조건부 집계: **`SUM(CASE WHEN 조건 THEN 1 ELSE 0 END)`** 공식을 외우자.
3. 비율 계산: **"특정 조건 수 / 전체 수"** 를 구할 때 `WHERE` 쓰지 말고 위 공식을 쓴다.

