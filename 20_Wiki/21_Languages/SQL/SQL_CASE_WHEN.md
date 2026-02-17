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
  - "[[Aggregation_GROUP_BY]]"
  - "[[Pivot_Unpivot]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
---
## 개념 한 줄 요약

SQL에서 **"만약(IF) ~라면 A, 아니면(ELSE) B"** 라는 논리를 구현하는 문법이야.
엑셀의 `IF` 함수나 프로그래밍의 `if-else` 문과 똑같아.

---
## 왜 필요한가 (Why)

1.  **데이터 범주화 (Categorization):** 점수(85점)를 등급('A학점')으로 바꿀 때.
2.  **데이터 정제 (Cleaning):** `M`, `F`로 된 성별을 `Male`, `Female`로 바꿀 때.
3.  **조건부 집계 (Conditional Aggregation):** (핵심🔥) 전체 매출 중 '전자제품' 매출만 따로 컬럼으로 뽑고 싶을 때.

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
        ELSE 'C학점'                    -- 나머지 전부
    END AS grade                        -- [문법 끝] 컬럼 별명(AS) 필수!
FROM exams;
```

---
## 데이터 엔지니어의 핵심 기술: 조건부 집계 (Conditional Aggregation) 

초보자는 `WHERE`를 쓰고, 고수는 `CASE WHEN`을 씁니다. 
특히 **"전체 중에 특정 조건의 비율(%)"** 을 구할 때 이 기술이 필수입니다.

### ❌ 초보자의 고민 (WHERE의 한계)

> "남자의 숫자도 세고 싶고, 여자의 숫자도 세고 싶은데... `WHERE gender = 'M'`을 하면 여자가 사라지고, `WHERE gender = 'F'`를 하면 남자가 사라져요!


### ⭕️ 해결책: `SUM` + `CASE WHEN` (피벗)

데이터를 버리지(`WHERE`) 말고, **살려둔 채로 표시만 다르게** 하는 겁니다.

- **원리:** 조건에 맞으면 **1**, 아니면 **0**을 부여하고 다 더해버림(`SUM`).

```sql
SELECT
    class_name,
    COUNT(*) AS total_students, -- 전체 학생 수
    
    -- [남자만 세기] 남자는 1, 여자는 0으로 바꿔서 합계 구하기
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    
    -- [여자만 세기] 여자는 1, 남자는 0으로 바꿔서 합계 구하기
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count

FROM students
GROUP BY class_name;
```

**결과:** 

| 반 | 전체 | 남자 | 여자 |
|:---:|:---:|:---:|:---:| 
| A반 | 30 | 10 | 20 |

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
## (PostgreSQL 전용) 더 우아한 문법: `FILTER` 

만약 **PostgreSQL**을 쓰고 있다면, `CASE WHEN` 대신 **`FILTER`** 구문을 쓸 수 있습니다. 
기능은 똑같은데 코드가 훨씬 짧고 **"아, 여기서 필터링해서 세는구나!"** 하고 바로 이해됩니다

### 비교 체험 (기프트카드 결제 비율 구하기)

**[1] 표준 SQL 방식 (`CASE WHEN`)**

```sql
SELECT
    -- 조건부 집계
    SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END) AS gift_count,
    COUNT(*) AS total_count
FROM payments;
```

**[2] PostgreSQL 방식 (`FILTER`) ⭐️**

> 집계 함수 바로 뒤에 `FILTER`를 붙여서 깔끔하게 해결!

```sql
SELECT
    -- "COUNT를 하긴 할 건데(COUNT*), 이 조건일 때만(FILTER) 세어줘"
    COUNT(*) FILTER (WHERE credit ILIKE '%gift%') AS gift_count,
    COUNT(*) AS total_count
FROM payments;
```


## BigQuery 전용) `FILTER`와 `COUNTIF` ☁️

BigQuery는 PostgreSQL의 `FILTER` 기능을 완벽하게 지원하며, 심지어 더 짧은 전용 함수도 있습니다.

### [1] FILTER 문법 (PostgreSQL과 동일)

BigQuery에서도 이 표준 문법이 그대로 작동합니다.

```sql
SELECT
    -- "기프트카드 결제 건수만 세어줘"
    COUNT(*) FILTER (WHERE credit LIKE '%gift%') AS gift_count
FROM `project.dataset.table`
```


### [2] BigQuery만의 필살기: `COUNTIF` 

`FILTER`도 길다고 생각했는지, 구글이 만든 초단축 함수입니다. 
**"조건(IF)에 맞는 것만 세라(COUNT)"** 는 뜻입니다.

```sql
SELECT
    -- 문법: COUNTIF(조건)
    COUNTIF(credit LIKE '%gift%') AS gift_count,
    
    -- 비율 계산할 때도 훨씬 깔끔함
    COUNTIF(credit LIKE '%gift%') / COUNT(*) AS gift_ratio
FROM `project.dataset.table`
```

## 요약표 (Dialect 비교)

|**기능**|**표준 SQL (ANSI)**|**PostgreSQL**|**BigQuery (GoogleSQL)**|
|---|---|---|---|
|**기본 (길다)**|`SUM(CASE WHEN ...)`|`SUM(CASE WHEN ...)`|`SUM(CASE WHEN ...)`|
|**중급 (깔끔)**|-|**`FILTER (WHERE ...)`**|**`FILTER (WHERE ...)`**|
|**고급 (초단축)**|-|-|**`COUNTIF(조건)`**|

>**결론:** BigQuery를 쓴다면 **`COUNTIF`** 가 가장 압도적으로 편합니다. 
>하지만 다른 DB(Postgres 등)와 호환성을 생각한다면 `FILTER`를 쓰는 습관을 들이는 게 좋습니다.

---
## 초보자가 자주 착각하는 포인트

- **Q. `ELSE`를 안 쓰면 어떻게 돼?**
    - 조건에 안 맞는 데이터는 자동으로 **`NULL`** 이 돼. 
    - 빈 값이 생기는 걸 원치 않으면 `ELSE 'Unknown'` 처럼 기본값을 꼭 넣어줘.

- **Q. `END`를 자꾸 까먹어.**
    - SQL에서 `CASE`를 열었으면 무조건 `END`로 닫아줘야 해. 괄호를 닫는 것과 같은 이치야.

---
## 요약

1. 단순 변환: **"점수 ➔ 등급"** 바꿀 때 쓴다.
2. 조건부 집계: **`SUM(CASE WHEN 조건 THEN 1 ELSE 0 END)`** 공식을 외우자.
3. 비율 계산: **"특정 조건 수 / 전체 수"** 를 구할 때 `WHERE` 쓰지 말고 위 공식을 쓴다.

