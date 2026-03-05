---
aliases:
  - 다중 테이블 조인
  - NON EQUI JOIN
  - EQUI JOIN
tags:
  - SQL
related:
  - "[[SQL_Standard_JOIN]]"
  - "[[SQL_ERD_Components]]"
  - "[[SQL_Keys_and_Identifiers]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Self_Join]]"
---


# SQL JOIN 개념

## 개념 한 줄 요약

> **"여러 테이블에 흩어진 데이터를 공통 컬럼을 기준으로 엮어내는 기술."** PK-FK 제약조건이 없어도 데이터의 의미(Domain)만 같으면 어떤 컬럼으로든 조인할 수 있다.

---

---

# ① JOIN 의 기본 원리 — 카테시안 곱

> **JOIN 은 두 테이블의 모든 행을 한 번씩 곱해보는 것(CROSS JOIN) 에서 출발한다.** 그 중에서 `ON` 조건에 맞는 행만 필터링해서 남긴다.

```
A 테이블 10줄 × B 테이블 10줄 = 100줄 (카테시안 곱)
                ↓ ON 조건 필터링
            조건에 맞는 행만 남김
```

```sql
-- Ambiguous 에러: dept_id 가 어느 테이블 소속인지 불명확
SELECT dept_id, emp_name FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- ✅ 정상: 테이블 별칭으로 소속 명시
SELECT e.dept_id, e.emp_name, d.dept_name FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

> **조인 쿼리에서는 모든 컬럼에 테이블 별칭을 붙이는 것을 강력히 권장한다.**

---

---

# ② EQUI JOIN — 등가 조인

> **조인 조건에 `{text}=` 연산자를 사용하는 방식.** 실무 조인 쿼리의 95% 이상이 EQUI JOIN 이다.

```sql
SELECT e.emp_name, e.dept_id, d.dept_name
FROM employees e
INNER JOIN departments d
    ON e.dept_id = d.dept_id;  -- = 연산자: 정확히 일치할 때 연결
```

```
사원 테이블의 dept_id = 부서 테이블의 dept_id
→ 값이 정확히 일치하는 행끼리 연결
```

## SELECT 절 대체 가능

```sql
-- EQUI JOIN 은 ON 조건의 양쪽 컬럼 값이 동일
-- → SELECT 절에서 e.dept_id 를 써도 d.dept_id 를 써도 결과 같음
SELECT e.dept_id   -- 또는 d.dept_id 둘 다 동일한 값
```

---

---

# ③ NON EQUI JOIN — 비등가 조인

> **조인 조건에 `{text}=` 이외의 연산자 (`>` `<` `BETWEEN` 등) 를 사용하는 방식.** 값이 정확히 일치하지 않고 **범위 안에 포함될 때** 사용한다.

```sql
SELECT e.emp_name, e.salary, s.grade
FROM employees e
INNER JOIN salary_grades s
    ON e.salary BETWEEN s.min_salary AND s.max_salary;
-- 급여가 등급표의 범위 안에 포함될 때 연결
```

```
사원 급여: 350만원
등급표:  min_salary=300 ~ max_salary=400  →  B등급

350만원 ≠ 300만원 (정확히 일치하지 않음)
350만원 이 300~400 범위 안에 포함됨  →  BETWEEN 사용
```

## ⚠️ SELECT 절 대체 불가

```sql
-- EQUI JOIN: e.id = d.id → 값이 같으므로 어느 쪽 써도 무관
-- NON EQUI JOIN: e.salary ≠ s.min_salary → 값이 다름!

SELECT s.grade       -- ✅ 등급은 salary_grades 테이블에서
SELECT e.salary      -- ✅ 실제 급여는 employees 테이블에서
SELECT s.min_salary  -- ❌ 최소 급여와 실제 급여는 다른 값
```

> NON EQUI JOIN 에서는 ON 조건에 쓰인 컬럼이라도 서로 값이 다르다. 어느 테이블의 컬럼을 SELECT 할지 반드시 명확히 구분해야 한다.

---

---

# ④ EQUI vs NON EQUI 비교

|구분|연산자|예시|특징|
|---|---|---|---|
|**EQUI JOIN**|`=`|`e.dept_id = d.dept_id`|값이 정확히 일치|
|**NON EQUI JOIN**|`>` `<` `BETWEEN` 등|`sal BETWEEN min AND max`|범위 안에 포함|

---

---

# ⑤ 다중 테이블 조인 — 항상 2개씩 처리

> **테이블이 3개 이상이어도 DB 는 항상 2개씩 순서대로 처리한다.**

```
① A JOIN B → 중간 결과 AB
② AB JOIN C → 최종 결과 ABC
```

```sql
SELECT e.emp_name, d.dept_name, s.grade
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id      -- ① A + B
INNER JOIN salary_grades s                              -- ② AB + C
    ON e.salary BETWEEN s.min_salary AND s.max_salary;
```

---

---

# 자주 하는 실수

## "EQUI JOIN = INNER JOIN, NON EQUI JOIN = OUTER JOIN 아닌가요?"

```
❌ 완전히 다른 분류 기준

EQUI / NON EQUI  →  조건식에 어떤 연산자를 썼느냐  (= vs BETWEEN 등)
INNER / OUTER    →  짝 못 찾은 행을 버리냐 살리냐

"EQUI 조건(=) 을 사용하면서 INNER JOIN 방식을 쓴다" 가 맞는 표현
서로 다른 차원의 분류 체계
```

## "PK/FK 없으면 조인 못 하나요?"

```
❌ 물리적 제약조건은 필수가 아님
값의 의미(Domain)만 같다면 어떤 컬럼으로든 조인 가능
ON 조건은 쿼리 작성 시점의 논리적 조건
```

## "EQUI JOIN 이랑 서브쿼리랑 헷갈려요"

```text
둘 다 두 테이블의 데이터를 엮는 것처럼 보이지만 완전히 다른 개념

JOIN       →  두 테이블을 옆으로 붙여서 컬럼을 늘린다 (가로 확장)
SubQuery   →  쿼리 안에 쿼리를 넣어서 조건이나 값으로 활용한다
```

```sql
-- JOIN: 사원 + 부서를 붙여서 부서명 컬럼을 함께 출력
SELECT e.emp_name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;

-- SubQuery: 부서 테이블을 조건 검색에만 활용 (결과에 부서 컬럼 없음)
SELECT emp_name
FROM employees
WHERE dept_id IN (SELECT dept_id FROM departments WHERE dept_name = '개발팀');
```

|구분|목적|결과|
|---|---|---|
|**JOIN**|두 테이블 컬럼을 함께 보고 싶을 때|컬럼이 늘어남 (가로 확장)|
|**SubQuery**|다른 테이블을 조건·값으로만 활용할 때|컬럼 수 그대로|

>EQUI / NON EQUI 는 JOIN 안에서 `{text}=` 을 쓰냐 `BETWEEN` 을 쓰냐의 차이일 뿐, 서브쿼리와는 아예 다른 개념이다.

---
