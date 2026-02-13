---
aliases:
  - 윈도우함수
  - 분석함수
  - RANK
  - LEAD
  - LAG
  - ROW_NUMBER
tags:
  - PyFlink
related:
  - "[[00_SQL_HomePage]]"
  - "[[00_Apache Flink_HomePage]]"
  - "[[PyFlink_Windows]]"
  - "[[SQL_JOIN_Concept]]"
---
##  개념 한 줄 요약

**"행(Row)을 압축하지 않고, 원래 그대로 두면서 통계(순위, 합계, 이전값)를 구하는 마법."**
(`GROUP BY`와의 결정적 차이: `GROUP BY`는 결과가 1줄로 요약되지만, 윈도우 함수는 원래 행 개수가 그대로 유지됨)

---
## 왜 필요한가? (Why)

**문제점:**
- "매출액 순으로 1등부터 10등까지 뽑고 싶은데, `ORDER BY`만 쓰면 순위 번호(1, 2, 3...)가 안 생긴다."
- "오늘 매출이 어제보다 올랐는지 알고 싶은데, SQL은 **'바로 윗줄'** 데이터를 못 쳐다본다." (Self Join을 해야 해서 쿼리가 엄청 복잡해짐)

**해결책:**
- **`RANK()`** 로 순위 번호를 자동으로 매긴다.
- **`LAG()`** 로 "내 바로 윗줄(어제)" 데이터를 옆으로 가져온다.

---
## 실무 적용 사례 (Practical Context)

1.  **순위 매기기:** "부서별로 연봉이 높은 상위 3명 뽑기" (`RANK`)
2.  **증감률 분석:** "이번 달 매출이 지난달 대비 몇 % 성장했나?" (`LAG`)
3.  **이동 평균:** "최근 3일간의 주가 평균 구하기" (`AVG`)
4.  **연속성 체크:** "사용자가 3일 연속으로 출석했는가?" (`LEAD`/`LAG` + 날짜계산)

---
##  Code Core Points: 문법 공식

> **핵심 문법:** `{sql}함수() OVER (PARTITION BY 그룹 ORDER BY 정렬)`

* **`PARTITION BY`**: 엑셀의 '부분합'처럼, 누구끼리 묶어서 계산할지 정함. (생략 가능)
* **`ORDER BY`**: 순위나 전후 관계를 따질 때 기준이 되는 정렬 순서.

### ① 순위 구하기 (Ranking)

```sql
-- 1. RANK(): 공동 1등이 2명이면, 다음은 3등 (1, 1, 3...)
RANK() OVER (ORDER BY salary DESC)

-- 2. DENSE_RANK(): 공동 1등이 있어도, 다음은 2등 (1, 1, 2...) 
DENSE_RANK() OVER (ORDER BY salary DESC)

-- 3. ROW_NUMBER(): 공동 등수 없음. 무조건 고유 번호 (1, 2, 3...)
ROW_NUMBER() OVER (ORDER BY salary DESC)
```

### ② 앞뒤 값 가져오기 (Offset) ⭐️ 가장 중요!

데이터 타입에 주의해야 합니다. 
날짜가 **문자(Text)** 로 되어있으면 계산할 때 에러가 납니다.

```sql
-- LAG: 내 앞의 행 (어제 값)
LAG(컬럼명, 1) OVER (ORDER BY 날짜)

-- LEAD: 내 뒤의 행 (내일 값)
LEAD(컬럼명, 1) OVER (ORDER BY 날짜)

-- [심화] 날짜 포맷 변환 (::date) ⭐️ 중요 
-- "Operator does not exist: text - text" 에러 해결법! 
-- 날짜가 글자('2022-01-25')로 저장되어 있다면, 반드시 ::date를 붙여야 날짜로 인식함.
LAG(컬럼명::date) OVER (ORDER BY 컬럼명::date)
```

---
## 상세 분석: 연속된 날짜 + 전날 값 비교 (PostgreSQL)

**상황:** `daily_sales` 테이블에서 매일의 매출 변화를 보고 싶다. 
단, 데이터가 빠진 날(휴무일 등)이 있을 수 있으니 **"진짜 어제"** 인지 확인해야 한다.

```sql
SELECT
    sales_date AS "오늘날짜",
    amount AS "오늘매출",
    
    -- 1. 어제 매출 가져오기 (LAG)
    -- sales_date가 혹시 글자(VARCHAR)일 수 있으니 안전하게 ::date를 붙임
    LAG(amount) OVER (ORDER BY sales_date::date) AS "전날매출",
    
    -- 2. 매출 증감액 (오늘 - 어제)
    amount - LAG(amount) OVER (ORDER BY sales_date::date) AS "증감액",
    
    -- 3. [핵심] 날짜 연속성 확인 (오늘 - 이전데이터날짜)
    -- 글자끼리는 뺄셈 불가! 반드시 ::date로 바꿔서 빼야 '일수(숫자)'가 나옴.
    sales_date::date - LAG(sales_date::date) OVER (ORDER BY sales_date::date) AS "날짜차이"

FROM daily_sales
ORDER BY sales_date::date;
```

**결과 해석표**

| **오늘날짜**   | **오늘매출** | **전날매출(LAG)** | **증감액** | **날짜차이** | **해석**                    |
| ---------- | -------- | ------------- | ------- | -------- | ------------------------- |
| 2024-01-01 | 100      | NULL          | NULL    | NULL     | 첫 데이터                     |
| 2024-01-02 | 150      | 100           | +50     | **1**    | **연속됨 (어제 대비 50 증가)**     |
| 2024-01-04 | 200      | 150           | +50     | **2**    | **데이터 끊김 (1월 3일 데이터 없음)** |

> **Tip:** `날짜차이`가 **1**이면 연속된 날짜이고, 1보다 크면 중간에 비어있는 날이 있다는 뜻입니다!

>"PostgreSQL에서 날짜 계산하다가 에러 나면 99%는 **데이터 타입** 문제입니다. 
>컬럼 뒤에 **`::date`** (날짜로 변신!)나 **`::int`** (숫자로 변신!)를 붙여주는 습관을 들이세요. 
>이걸 **'캐스팅(Casting)'** 이라고 합니다."

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "윈도우 함수 결과를 `WHERE` 절에서 바로 쓸 수 있나요?"

- **절대 불가 (X)**. `SELECT`에서 만들어진 별명(Alias)이나 윈도우 함수는 `WHERE` 절에서 안 보입니다.
- **해결:** 서브쿼리(Subquery)나 CTE(`WITH` 문)로 한 번 감싸야 합니다.

```sql
-- [틀린 예시] 1등만 뽑고 싶어!
SELECT name, RANK() OVER(...) as rnk FROM table WHERE rnk = 1; (에러!)

-- [맞는 예시]
WITH ranking_data AS (
    SELECT name, RANK() OVER(...) as rnk FROM table
)
SELECT * FROM ranking_data WHERE rnk = 1;
```

### ② "`PARTITION BY`를 빼먹으면 어떻게 되나요?"

- 테이블 **전체**를 하나의 그룹으로 봅니다.
- "부서별 순위"를 구해야 하는데 `PARTITION BY department`를 빼면, "전 직원 통합 순위"가 나와버립니다.

### ③ "CTE 안에 WHERE 절을 미리 넣으면 안 되나요?"

- **가장 많이 하는 실수입니다!** 
- CTE 안에서 `WHERE pm10 >= 30`을 먼저 해버리면, 30 미만이었던 날짜들이 사라져서 `LAG`가 "진짜 어제"가 아닌 "그저께"나 "지난주" 데이터를 가져오게 됩니다. 
- **윈도우 함수를 쓸 때는 전체 데이터를 먼저 계산하고 밖에서 필터링**하는 것이 철칙입니다.

### ④ "`LAG(date_col, 2)`를 쓰면 '2일 전' 날짜가 구해지나요?" 

- **아니요! (이게 제일 헷갈림)**
- `LAG(measured_at, 2)`는 **"2일 전(시간)"** 을 계산하는 게 아니라, **"위로 2칸(행) 위에 있는 값"** 을 가져오는 것입니다.
- 만약 데이터가 매일 있지 않고 띄엄띄엄 있다면(예: 1월 1일, 1월 5일, 1월 10일), `LAG(..., 1)`은 '어제'가 아니라 '5일 전' 데이터를 가져올 수도 있습니다.
- **해결책:** "진짜 2일 전인가?"를 확인하려면, 일단 `LAG`로 값을 가져온 뒤 **`오늘날짜 - LAG로가져온날짜 = 2`** 인지 뺄셈 계산을 따로 해야 합니다.