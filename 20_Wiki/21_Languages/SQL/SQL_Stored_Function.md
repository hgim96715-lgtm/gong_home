---
aliases:
  - 저장함수
  - plpgsql
  - CREATE FUNCTION
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DDL_Create]]"
  - "[[SQL_Top_N_Query]]"
---

# SQL_Stored_Function — 저장 함수 (PostgreSQL)

## 한 줄 요약

```
저장 함수 = SQL 로직을 함수로 만들어 재사용
CREATE FUNCTION + plpgsql 로 작성
RETURN QUERY 로 결과 집합 반환
```

---

---

# ① 기본 구조 ⭐️

```sql
CREATE OR REPLACE FUNCTION 함수명(파라미터 타입)
RETURNS TABLE (컬럼명 타입) AS $$
BEGIN
    -- 로직
    RETURN QUERY
        SELECT ...;
END;
$$ LANGUAGE plpgsql;
```

```
CREATE OR REPLACE:
  없으면 새로 생성
  있으면 덮어씌움 (매번 DROP 안 해도 됨)

RETURNS TABLE (...):
  행 집합을 반환할 때 사용
  컬럼명과 타입을 명시

$$ ... $$:
  함수 본문을 감싸는 달러 인용 (Dollar Quoting)
  안에 작은따옴표 자유롭게 사용 가능

LANGUAGE plpgsql:
  PostgreSQL 전용 절차형 언어
  IF / LOOP / EXCEPTION 등 사용 가능
```

---

---

# ② RETURNS TABLE vs RETURNS SETOF

```sql
-- RETURNS TABLE: 컬럼명 직접 지정
CREATE FUNCTION get_top(n INT)
RETURNS TABLE (salary INT) AS $$
...

-- RETURNS SETOF: 기존 타입 재사용
CREATE FUNCTION get_employees()
RETURNS SETOF employees AS $$
...

-- RETURNS 단일값
CREATE FUNCTION get_count()
RETURNS INT AS $$
...
```

---

---

# ③ IF / THEN / END IF ⭐️

```sql
-- 기본 구조
IF 조건 THEN
    -- 참일 때
END IF;

-- IF / ELSE
IF 조건 THEN
    -- 참
ELSE
    -- 거짓
END IF;

-- IF / ELSIF / ELSE
IF 조건1 THEN
    -- 조건1 참
ELSIF 조건2 THEN
    -- 조건2 참
ELSE
    -- 전부 거짓
END IF;
```

## 실전 예시 — N이 1 미만이면 NULL 반환

```sql
CREATE OR REPLACE FUNCTION NthHighestSalary(N INT)
RETURNS TABLE (Salary INT) AS $$
BEGIN
    -- N 이 1 미만이면 NULL 반환하고 함수 종료
    IF N < 1 THEN
        RETURN QUERY SELECT NULL::INT;
        RETURN;                          -- 함수 즉시 종료
    END IF;

    -- 정상 케이스
    RETURN QUERY (
        SELECT DISTINCT e.salary
        FROM Employee e
        ORDER BY e.salary DESC
        OFFSET N - 1
        LIMIT 1
    );
END;
$$ LANGUAGE plpgsql;
```

```
RETURN QUERY SELECT NULL::INT:
  NULL 을 INT 타입으로 캐스팅해서 반환
  RETURNS TABLE (Salary INT) 와 타입 맞춰야 함

RETURN (단독):
  함수 즉시 종료 (조기 탈출)
  RETURN QUERY 와 다름 — 값 없이 그냥 종료
```

---

---

# ④ RETURN QUERY ⭐️

```sql
-- SELECT 결과를 그대로 반환
RETURN QUERY
    SELECT id, name FROM users WHERE active = true;

-- 서브쿼리도 가능
RETURN QUERY (
    SELECT DISTINCT salary
    FROM Employee
    ORDER BY salary DESC
    OFFSET N - 1
    LIMIT 1
);
```

```
RETURN QUERY vs RETURN:
  RETURN QUERY SELECT ...  → 쿼리 결과를 반환
  RETURN                   → 함수 즉시 종료 (반환값 없음)

RETURN QUERY 를 여러 번 쓰면?
  결과가 누적되어 전부 반환됨
```

---

---

# ⑤ N번째 높은 값 찾기 패턴 ⭐️

## MySQL vs PostgreSQL 문법 차이

```
MySQL:
  RETURN QUERY 없음
  RETURN (SELECT ...) 형태
  RETURNS INT (단일값 반환)

PostgreSQL:
  RETURN QUERY 사용
  RETURNS TABLE (...) 로 집합 반환
  plpgsql 언어
```

## PostgreSQL — DENSE_RANK 방식 (추천) ⭐️

```sql
CREATE OR REPLACE FUNCTION NthHighestSalary(N INT)
RETURNS TABLE (salary_out INT) AS $$
BEGIN
    RETURN QUERY (
        WITH rnk_salary AS (
            SELECT DISTINCT salary,
                   DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
            FROM Employee
        )
        SELECT MAX(rnk_salary.salary)
        FROM rnk_salary
        WHERE rnk = N
    );
END;
$$ LANGUAGE plpgsql;
```

## MySQL — DENSE_RANK 방식


```sql
CREATE FUNCTION getNthHighestSalary(N INT)
RETURNS INT
BEGIN
    RETURN (
        WITH rnk_salary AS (
            SELECT DISTINCT salary,
                   DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
            FROM Employee
        )
        SELECT salary
        FROM rnk_salary
        WHERE rnk = N
    );
END;
```

## MySQL vs PostgreSQL 비교표

|구분|MySQL|PostgreSQL|
|---|---|---|
|반환 방식|`RETURN (SELECT ...)`|`RETURN QUERY (SELECT ...)`|
|반환 타입|`RETURNS INT` (단일값)|`RETURNS TABLE (컬럼 타입)`|
|언어|SQL|`LANGUAGE plpgsql`|
|CTE|`WITH ...` 가능|`WITH ...` 가능|
|함수 본문|`BEGIN RETURN (...); END`|`BEGIN RETURN QUERY (...); END`|

```
핵심 차이:
  MySQL  → RETURN (쿼리)      단일 RETURN 문에 쿼리 넣음
  PostgreSQL → RETURN QUERY (쿼리)  QUERY 키워드 필수

  MySQL  → RETURNS INT       단일 값
  PostgreSQL → RETURNS TABLE (salary_out INT)  행 집합
```

## DENSE_RANK vs OFFSET 비교


```sql
-- ① OFFSET + LIMIT (구식)
SELECT DISTINCT salary
FROM Employee
ORDER BY salary DESC
OFFSET N - 1
LIMIT 1;

-- ② DENSE_RANK (요즘 추천) ⭐️
WITH rnk_salary AS (
    SELECT DISTINCT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM Employee
)
SELECT MAX(salary)
FROM rnk_salary
WHERE rnk = N;
```

```
OFFSET 방식의 문제:
  데이터가 없는 N이면 빈 결과
  가독성 낮음

DENSE_RANK 방식의 장점:
  동순위 처리가 명확
  NULL 반환 처리가 자연스러움 (결과 없으면 NULL)
  코드 의도가 명확히 보임
  요즘 면접 / 코딩테스트에서 선호
```

## 다른 방법들 비교


```sql
-- ③ ROW_NUMBER (동순위 고려 안 함)
WITH ranked AS (
    SELECT salary,
           ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM (SELECT DISTINCT salary FROM Employee) t
)
SELECT salary FROM ranked WHERE rn = N;

-- ④ 서브쿼리 중첩 (2번째만 가능 / N 일반화 어려움)
SELECT MAX(salary)
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

```
RANK vs DENSE_RANK vs ROW_NUMBER:
  100, 100, 90 이 있을 때

  RANK:         1, 1, 3   (동순위 다음 건너뜀)
  DENSE_RANK:   1, 1, 2   (동순위 다음 연속)  ← N번째에 적합
  ROW_NUMBER:   1, 2, 3   (무조건 고유)
```

---

---

# ⑥ 함수 호출 & 관리

```sql
-- 함수 호출
SELECT * FROM NthHighestSalary(3);
SELECT NthHighestSalary(3);

-- 함수 목록 확인
SELECT routine_name, routine_type
FROM information_schema.routines
WHERE routine_schema = 'public';

-- 함수 삭제
DROP FUNCTION IF EXISTS NthHighestSalary(INT);

-- 함수 내용 확인
SELECT pg_get_functiondef('NthHighestSalary'::regproc);
```

---

---

# ⑦ NULL::타입 캐스팅

```sql
-- NULL 을 특정 타입으로 캐스팅
SELECT NULL::INT        -- NULL (정수형)
SELECT NULL::TEXT       -- NULL (문자형)
SELECT NULL::DATE       -- NULL (날짜형)

-- RETURNS TABLE (Salary INT) 와 타입 일치시킬 때 필수
RETURN QUERY SELECT NULL::INT;
-- NULL 만 쓰면 타입 불명확 → 에러 가능
```

---

---

# ⑧ 실전 패턴 모음

## 파라미터 유효성 검사

```sql
CREATE OR REPLACE FUNCTION get_page(page_no INT, page_size INT)
RETURNS TABLE (id INT, name TEXT) AS $$
BEGIN
    -- 유효성 검사
    IF page_no < 1 THEN
        RAISE EXCEPTION 'page_no must be >= 1, got %', page_no;
    END IF;

    IF page_size < 1 OR page_size > 1000 THEN
        RAISE EXCEPTION 'page_size must be 1~1000';
    END IF;

    RETURN QUERY
        SELECT u.id, u.name
        FROM users u
        ORDER BY u.id
        OFFSET (page_no - 1) * page_size
        LIMIT page_size;
END;
$$ LANGUAGE plpgsql;
```

## 조건에 따라 다른 쿼리 실행

```sql
CREATE OR REPLACE FUNCTION get_salary_stats(dept TEXT)
RETURNS TABLE (avg_sal NUMERIC, max_sal INT, min_sal INT) AS $$
BEGIN
    IF dept = 'ALL' THEN
        RETURN QUERY
            SELECT AVG(salary)::NUMERIC, MAX(salary), MIN(salary)
            FROM Employee;
    ELSE
        RETURN QUERY
            SELECT AVG(salary)::NUMERIC, MAX(salary), MIN(salary)
            FROM Employee
            WHERE department = dept;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

---

# 자주 하는 실수

| 실수                 | 원인                  | 해결                           |
| ------------------ | ------------------- | ---------------------------- |
| `RETURN` 뒤에도 실행됨   | `RETURN QUERY` 와 혼동 | `RETURN` 단독 = 즉시 종료          |
| NULL 반환 타입 에러      | 타입 미지정              | `NULL::INT` 처럼 타입 캐스팅        |
| OFFSET 0 인데 2번째 나옴 | DISTINCT 없음         | `SELECT DISTINCT` 추가         |
| 함수 수정 후 반영 안 됨     | `OR REPLACE` 없음     | `CREATE OR REPLACE FUNCTION` |
| `END IF` 없음        | 문법 오류               | `IF ... THEN ... END IF;`    |
| `$$` 닫는 거 빠짐       | 문법 오류               | 함수 끝에 `$$ LANGUAGE plpgsql;` |