---
aliases:
  - 서브쿼리
  - CTE
  - WITH절
  - 임시테이블
  - 가독성
tags:
  - SQL
related:
  - "[[Window_Functions]]"
---
## 개념 한 줄 요약

**"쿼리 안에 또 다른 쿼리를 넣는 기술."**

* **SubQuery (서브쿼리):** 괄호 `( )` 안에 쿼리를 구겨 넣는 방식. (마트료시카 인형처럼 계속 들어감) 
* **CTE (WITH 절):** 쿼리 덩어리에 **이름표**를 붙여서 맨 위에 따로 빼두는 방식. (레고 블록처럼 미리 만들어두고 조립함) 

---

##  왜 필요한가? (Why)

**문제점:**
- "평균 연봉보다 더 많이 받는 직원을 찾고 싶어요."
- -> 순서상 **'평균 연봉'** 을 먼저 구해야, 누구랑 비교를 하든지 말든지 한다.
- -> 즉, **쿼리 실행 도중에 '임시 결과'가 필요**하다.

**해결책:**
- 괄호 안에 `SELECT AVG(salary)...`를 넣어서(서브쿼리) 해결한다.
- 하지만 괄호가 3중, 4중으로 겹치면 **해석 불가(Spaghetti Code)** 가 된다.
- **CTE**를 써서 "평균연봉 구하는 블록", "필터링 블록"을 따로 정의해서 가독성을 높인다.

---

## 실무 적용 사례 (Practical Context)

1.  **복잡한 단계별 집계:**
    * (1단계) 일단 이번 달 결제 내역만 추린다.
    * (2단계) 추린 데이터로 고객별 총액을 구한다.
    * (3단계) 그중 VIP 등급만 뽑는다.
    * -> 이걸 한 번에 짜면 망한다. CTE로 1, 2, 3단계를 나눈다.
2.  **재사용:**
    * "아까 구한 '평균 연봉'을 여기서도 쓰고, 저기서도 써야 해."
    * CTE로 한 번 만들어두면 밑에서 여러 번 불러쓸 수 있다.

---

##  Code Core Points: 서브쿼리 vs CTE

**상황:** "부서별 평균 연봉보다 더 많이 받는 직원 찾기"

### ① SubQuery (Old Style) 

괄호 안에 쿼리가 들어간다. 읽을 때 **안쪽부터 해석해서 바깥쪽으로** 나와야 한다. (직관적이지 않음)

```sql
SELECT emp_name, salary
FROM employees
WHERE salary > (
    -- 괄호 안에 갇혀있는 서브쿼리
    -- "이거 먼저 해석하고 나가세요"
    SELECT AVG(salary) FROM employees
);
```

### ② CTE (Modern Style - WITH) 

**`WITH 이름 AS (...)`** 형식을 쓴다. 위에서 아래로 물 흐르듯이 읽힌다.

```sql
-- 1단계 블록: 'avg_sal'이라는 이름표를 붙임
WITH avg_sal_block AS (
    SELECT AVG(salary) as avg_val FROM employees
),
-- 2단계 블록: (필요하면 콤마 찍고 계속 만들 수 있음)
high_earner_block AS (
    SELECT * FROM employees
)

-- 최종 조립 (Main Query)
SELECT emp_name, salary
FROM high_earner_block
WHERE salary > (SELECT avg_val FROM avg_sal_block);
```

---
## 상세 분석: CTE와 메인 쿼리 연결하기 (The Missing Link)

내가 헷갈려 했던 부분
CTE를 만들고 나서 막상 `SELECT`를 하려니 멍해진다면, 딱 **두 가지 규칙**만 기억하세요.

### 규칙 1. CTE 이름 = "새로 만든 테이블 이름"

CTE를 정의(`AS (...)`)하고 괄호를 닫는 순간, 그 이름은 **DB에 원래 있던 테이블**과 똑같은 취급을 받습니다.

- `FROM employees` 하듯이 **`FROM avg_sal_block`** 이라고 쓰면 됩니다.

### 규칙 2. "수출(Export)"해야 "수입(Import)"할 수 있다.

가장 중요한 점입니다. 
**CTE 내부에서 `SELECT` 하지 않은 컬럼은 메인 쿼리에서 절대 못 씁니다.**

#### [상황 예시]

1. CTE(`my_cte`) 안에서 `id`와 `name`만 `SELECT` 했다.
2. 그런데 메인 쿼리에서 `SELECT age FROM my_cte`를 요청했다?
3. 👉 **에러 발생!** (CTE 안에 `age`가 없으니까요.)

### 코드로 보는 연결 공식 (Connection Pattern)

**목표:** 부서별 평균 연봉을 구하고(CTE), 그 평균보다 많이 받는 사람 찾기(Main).

```sql
WITH dept_avg AS (
    -- [1] CTE 정의 단계 (수출 준비)
    -- 여기서 'dept_no'랑 'avg_salary' 두 개를 뽑았죠?
    -- 이제 이 두 컬럼만 바깥 세상에서 쓸 수 있습니다.
    SELECT dept_no, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_no
)

-- [2] 최종 조립 단계 (수입 및 활용)
SELECT 
    e.emp_name, 
    e.salary, 
    d.avg_salary  -- CTE에서 만든 컬럼을 가져옴!
FROM employees e
-- [핵심] CTE 이름을 진짜 테이블처럼 JOIN에 사용
JOIN dept_avg d ON e.dept_no = d.dept_no 
WHERE e.salary > d.avg_salary;
```

### 헷갈림 포인트 해결 (Q&A)

**Q. "CTE를 `FROM` 절에 써야 해요, `JOIN`에 써야 해요?"**
- **A.** 둘 다 됩니다!
    - CTE 데이터만 보고 싶으면: `FROM dept_avg`
    - 원래 테이블이랑 섞어서 보고 싶으면: `JOIN dept_avg ON ...` (위의 예시)

**Q. "CTE 안에서 컬럼 별명(`AS`)을 지어줬으면 어떻게 불러요?"**
- **A.** 무조건 **그 별명**으로 불러야 합니다.
    - CTE 안에서 `AVG(salary) AS avg_val` 이라고 했으면,
    - 바깥에서는 `avg_val` 이라고 찾아야지, `AVG(salary)`라고 찾으면 못 알아듣습니다.

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "CTE는 진짜 테이블인가요?"

- **아니요.** 쿼리가 실행되는 그 순간에만 잠시 메모리에 존재하는 **'임시 가상 테이블'** 입니다.
- 쿼리가 끝나면 흔적도 없이 사라집니다. (DB에 저장 안 됨)

### ② "서브쿼리는 절대 쓰면 안 되나요?"

- **아니요.** `WHERE id IN (SELECT id ...)` 처럼 아주 단순한 필터링에는 서브쿼리가 더 편할 때가 많습니다.
- **복잡한 로직**이나 **테이블 조인**이 필요할 때 CTE를 쓰라는 뜻입니다.

### ③ "CTE 정의하고 안 쓰면 에러 나나요?" (PostgreSQL)

- 네, 보통 정의했으면 밑에서 써줘야 합니다. 하지만 에러까진 아니고 그냥 무시되기도 합니다.
- 주의: `WITH` 문은 반드시 `SELECT`, `INSERT`, `UPDATE`, `DELETE` 같은 메인 쿼리 바로 위에 붙어있어야 합니다.

---
## PostgreSQL 특화: CTE 꿀팁 (Materialized) 

PostgreSQL은 CTE를 아주 똑똑하게 다룹니다.

```sql
WITH expensive_calculation AS NOT MATERIALIZED (
   -- 엄청 복잡한 계산
)
SELECT * ...
```

- 원래 CTE는 최적화를 위해 메인 쿼리에 병합되기도 하는데,
- **`MATERIALIZED`** (기본값인 경우 많음): "이 블록은 무조건 먼저 딱 한 번만 계산해서 결과를 저장해놔!"라고 강제할 수 있습니다. (성능 튜닝할 때 씀)

