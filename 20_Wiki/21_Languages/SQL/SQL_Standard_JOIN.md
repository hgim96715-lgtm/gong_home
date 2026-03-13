---
aliases:
  - join
  - inner JOIN
  - OUTER JOIN
  - CROSS JOIN
  - NATURAL JOIN
tags:
  - SQL
related:
  - "[[SQL_JOIN_Concept]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[SQL_Self_Join]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_UNION]]"
---
# SQL Standard JOIN

## 개념 한 줄 요약

> **"공통된 열(Key)을 기준으로 두 테이블을 하나로 합치는 연산이다."** 엑셀의 `VLOOKUP` 을 더 강력하게 쓰는 것과 같으며, 벤다이어그램으로 이해하면 가장 직관적이다.

> [!notice] 🚨 `JOIN` 으로 데이터를 아무리 살려두더라도, 이후에 실행되는 `WHERE` 절에서 조건을 걸면 `LEFT` · `FULL` · `INNER` 상관없이 조건에 맞지 않는 행은 모두 제거된다.

---

## 왜 필요한가?

데이터베이스는 중복을 피하기 위해 `회원정보` 와 `주문내역` 을 따로 저장한다. 분석하려면 **연결고리(Key)** 로 묶어서 하나의 표로 만들어야 한다.

**데이터 엔지니어가 상황별로 쓰는 JOIN:**

|상황|JOIN 종류|
|---|---|
|주문 이력이 있는 진성 고객만|INNER JOIN|
|주문 안 했어도 전체 고객 리스트|LEFT JOIN|
|요일별 가장 매출이 높은 결제 건|다중 조건 JOIN|
|성능 테스트용 더미 데이터 뻥튀기|CROSS JOIN|

---

---

#  핵심 개념 — `ON` 절 vs `WHERE` 절

> **"가장 많이 헷갈리는 부분, 여기서 한 번에 정리한다."**

## 한 줄 원칙

```
기준 테이블(FROM) 의 필터  →  WHERE 절
조인 대상 테이블의 필터    →  ON 절
```

## 왜 다른가?

|구분|실행 시점|하는 일|
|---|---|---|
|`ON` 절|테이블을 **엮는 순간**|조건 불일치 → 오른쪽을 NULL로 만든다 (왼쪽은 보존)|
|`WHERE` 절|엮은 결과 전체에서|조건 불일치 → **행 자체를 삭제** 한다|

## 코드로 비교

```sql
-- ✅ 올바른 패턴: INNER 쪽(B) 조건은 ON 절에
-- → A 데이터가 보존되고, B가 없으면 NULL로 채워짐
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON  A.id = B.customer_id
    AND B.status = 'DONE';    -- ✅ ON 절 → B 없어도 A는 살아있음

-- ❌ 잘못된 패턴: INNER 쪽(B) 조건을 WHERE 절에
-- → B가 NULL인 행이 전부 제거 → 사실상 INNER JOIN과 동일해짐
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON A.id = B.customer_id
WHERE B.status = 'DONE';      -- ❌ WHERE 절 → LEFT JOIN 의미 없어짐
```

|조건 위치|결과|
|---|---|
|INNER 쪽 조건 → **`ON` 절**|✅ A 데이터 보존, B 없으면 NULL 유지|
|INNER 쪽 조건 → **`WHERE` 절**|❌ NULL 행 제거 → 사실상 INNER JOIN|

## 판단 체크리스트

```
조건을 걸 때마다 이렇게 물어보자:

Q. 이 조건이 걸리면 FROM 테이블(기준)의 행이 사라져도 괜찮아?
   ├── YES → WHERE 절  (기준 테이블 자체를 필터링하는 조건)
   └── NO  → ON 절     (조인 대상을 필터링하되 기준은 보존해야 함)
```

---

---

# JOIN 종류별 정리

## ① INNER JOIN — 교집합 ⭐️ 가장 많이 씀

> "양쪽 모두에 데이터가 있는 완벽한 짝만 남긴다."

벤다이어그램: 두 원이 겹치는 **가운데** 만 색칠. 짝이 없는 데이터는 버려진다 → 결과 행 수가 줄어들 수 있다.

```sql
SELECT A.customer_name, B.order_date
FROM Customer A
INNER JOIN Orders B
    ON A.id = B.customer_id;
```

> **예시:** 회원 + 주문 INNER JOIN → 주문 내역이 있는 회원만 남음. 가입만 하고 주문 안 한 회원은 사라짐.

---

## ② LEFT JOIN — 왼쪽 기준 다 살리기 ⭐️ 실무에서 가장 많이 씀

> "왼쪽 테이블은 무조건 다 살리고, 오른쪽은 없으면 NULL로 둔다."

벤다이어그램: **왼쪽 원 전체** 색칠 (겹치는 부분 포함).

```sql
SELECT A.customer_name, B.order_date
FROM Customer A          -- A가 기준(주인공)
LEFT JOIN Orders B
    ON A.id = B.customer_id;
-- 주문 안 한 회원도 NULL 상태로 전부 조회됨
```

### 자주 하는 실수 — "없는 걸 찾을 때 INNER JOIN 쓰기"

```sql
-- ❌ 잘못된 접근: 조인 시점에 작품 없는 작가가 이미 제거됨
SELECT a.author_name
FROM Authors a
INNER JOIN Books b ON a.id = b.author_id
WHERE COUNT(b.id) = 0;  -- 이미 날아간 데이터는 되살릴 수 없다

-- ✅ 올바른 접근: LEFT JOIN으로 먼저 다 살린 뒤 NULL 필터링
SELECT a.author_name
FROM Authors a
LEFT JOIN Books b ON a.id = b.author_id
WHERE b.id IS NULL;     -- 매칭된 책이 없는 작가만
```

---

## ③ RIGHT JOIN — 오른쪽 기준 다 살리기

> "LEFT JOIN 과 방향만 반대."

**실무 팁:** 사람의 시선(왼쪽→오른쪽)과 맞지 않아 읽기 어렵다. 실무에서는 **테이블 위치를 바꿔서 무조건 LEFT JOIN으로 통일** 하는 것이 암묵적인 룰.

---

## ④ FULL OUTER JOIN — 합집합

> "너랑 나랑 가진 거 다 합치자. 없는 건 NULL로 두고."

벤다이어그램: **두 원 전체** 색칠.

- MySQL은 지원하지 않음 → `LEFT JOIN` + `RIGHT JOIN` 을 `UNION` 해서 씀
- 실무 사용 빈도 낮음

---

## ⑤ CROSS JOIN — 카테시안 곱

> "조건 없이 모든 경우의 수를 다 곱해버린다."

`ON` 절이 없음. **행(Row) 수가 곱하기** 로 폭발한다.

```
상의 2건 × 하의 3건 = 총 6벌의 모든 코디 조합
A테이블 1,000건 × B테이블 1,000건 = 순식간에 100만 건
```

- **열(Column):** A컬럼 3개 + B컬럼 4개 = 7개 (더하기)
- **행(Row):** A건수 × B건수 (곱하기)

```sql
-- 두 문법은 같은 결과
SELECT BOY_NAME, GIRL_NAME FROM GIRL, BOY;
SELECT BOY_NAME, GIRL_NAME FROM GIRL CROSS JOIN BOY;
```

> **실무 용도:** 더미 데이터 대량 생성할 때만 사용. 일반 비즈니스 로직에는 거의 안 씀.

---

## ⑥ NATURAL JOIN & USING

> "DB가 이름 같은 컬럼을 알아서 찾아 엮어준다." — `ON` 절 사용 불가

---

### NATURAL JOIN

두 테이블에서 **이름이 동일한 컬럼을 자동으로 전부 찾아** 조인한다. 단, 이름이 같은 **모든** 컬럼의 값이 **100% 동일한 행** 만 결과에 나온다.

```sql
SELECT * FROM 사원 NATURAL JOIN 부서;
-- 이름이 같은 컬럼(예: dept_id, status, updated_at 등)을 DB가 알아서 전부 조인 기준으로 사용
```

**NATURAL JOIN 이 작동하는 조건:**

```
테이블 A: dept_id = 10,  name = '홍길동', status = 'active'
테이블 B: dept_id = 10,  name = '개발팀', status = 'active'

→ dept_id, status 이름이 같고 값도 모두 일치 → 조인 성공
→ 만약 status 값이 하나라도 다르면 → 해당 행은 결과에서 통째로 제외
```

> ⚠️ **시한폭탄:** 나중에 누군가 같은 이름의 컬럼(`updated_at`, `status` 등)을 추가하는 순간 기존 쿼리가 소리소문없이 망가진다. **→ 실무에서는 절대 사용 금지**

---

### NATURAL JOIN · USING — 테이블명·별칭 표기 금지 ⭐️

> **NATURAL JOIN 과 USING 으로 조인한 컬럼은 SELECT 에서 테이블명이나 별칭을 붙이면 에러가 난다.**

이유: DB 가 두 테이블의 해당 컬럼을 **하나로 합쳐서 단일 컬럼으로 처리** 하기 때문에, 어느 테이블 소속인지 구분이 없어진다.

```sql
-- NATURAL JOIN 예시
SELECT A.dept_id  -- ❌ 에러! dept_id 는 A·B 합쳐진 단일 컬럼
FROM 사원 A NATURAL JOIN 부서 B;

SELECT dept_id    -- ✅ 테이블명 없이 그냥 써야 한다
FROM 사원 A NATURAL JOIN 부서 B;

-- USING 예시
SELECT A.id       -- ❌ 에러! USING 으로 지정된 컬럼에 별칭 금지
FROM Orders A
JOIN Payments B USING(id);

SELECT id         -- ✅ 테이블명·별칭 없이 컬럼명만
FROM Orders A
JOIN Payments B USING(id);

SELECT A.amount   -- ✅ USING 으로 지정되지 않은 컬럼은 별칭 가능
FROM Orders A
JOIN Payments B USING(id);
```

---

### USING(컬럼명) — ON 절의 명시적 축약형

조인할 컬럼을 직접 지정하므로 NATURAL JOIN 보다 안전하다. 단, 조인 기준 컬럼의 이름이 **양쪽 테이블에서 동일** 해야 사용 가능하다.

```sql
-- ON 절 방식
JOIN Orders B ON A.id = B.id

-- USING 축약 방식
JOIN Orders B USING(id)
-- 의미: A.id = B.id 와 완전히 동일하지만, id 컬럼이 결과에 하나만 나온다
```

---

### 세 방식 비교

|구분|조인 기준|별칭 사용|안전성|
|---|---|---|---|
|`NATURAL JOIN`|이름 같은 컬럼 **전부 자동**|❌ 조인 컬럼에 별칭 불가|❌ 위험|
|`USING(id)`|내가 **지정한 컬럼만**|❌ 조인 컬럼에 별칭 불가|✅ 안전|
|`ON A.id = B.id`|명시적 지정|✅ 모든 컬럼 별칭 가능|✅ 가장 안전|

---

---

# 핵심 원리 — JOIN 후 행 수가 늘어나는 이유

> **"가장 많이 터지는 사고 — 데이터 뻥튀기(Data Explosion)"**

JOIN 결과 행 수 = **왼쪽 매칭 건수 × 오른쪽 매칭 건수**

**예시: LEFT JOIN**

```
왼쪽 테이블 (회원 ID):  1,  3,  5
오른쪽 테이블 (주문 ID): 2,  3,  3   ← 3번 회원이 2번 주문

결과:
ID 1 → 오른쪽 없음           → [1, NULL]      1줄
ID 3 → 오른쪽에 3이 2개      → [3, 3], [3, 3] 2줄  ← 뻥튀기!
ID 5 → 오른쪽 없음           → [5, NULL]      1줄

최종: 1, 3, 3, 5  (총 4줄)  ← 예상했던 3줄이 아님!
```

항상 **1:N 관계를 파악** 하고 조인해야 한다.

---

---

# 레거시 → ANSI 변환 ⭐️ SQLD 빈출

Oracle `(+)` 문법을 ANSI 표준으로 변환할 때, **INNER 쪽 테이블(NULL이 될 수 있는 쪽) 의 조건은 반드시 `ON` 절에** 위치시켜야 한다.

```sql
-- 🚨 레거시: Oracle (+) 방식
SELECT A.customer_name, B.order_date
FROM Customer A, Orders B
WHERE A.id = B.customer_id(+)
  AND B.status(+) = 'DONE';   -- INNER 쪽(B) 조건에 (+) 붙임

-- ✅ ANSI 변환: INNER 쪽 조건 → ON 절로 이동
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON  A.id = B.customer_id
    AND B.status = 'DONE';    -- ✅ ON 절에 위치

-- ❌ 잘못된 변환: WHERE 절에 두면 INNER JOIN과 동일
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON A.id = B.customer_id
WHERE B.status = 'DONE';      -- ❌ B가 NULL인 행 전부 제거
```

---

---

# 고급 패턴

## 패턴 A — 다중 조건 JOIN (Groupwise Max)

> "요일별로 가장 비싼 결제 내역을 찾아라."

`ON` 절에 `AND` 를 써서 두 조건을 동시에 건다.

```sql
SELECT t.*
FROM tips t
JOIN (
    -- 1단계: 요일별 최고액 정답지를 먼저 구함
    SELECT day, MAX(total_bill) AS max_bill
    FROM tips
    GROUP BY day
) m
  ON  t.day = m.day              -- 조건 1: 요일이 같아야 하고
  AND t.total_bill = m.max_bill; -- 조건 2: 금액이 최고액과 같아야 함
```

---

## 패턴 B — 체인 JOIN (다중 테이블 연결)

> "연결고리만 있으면 몇 개든 이어붙인다."

```sql
-- 흐름: payment → rental → inventory → film_actor → actor
SELECT
    a.first_name,
    a.last_name,
    SUM(p.amount) AS total_revenue
FROM payment p
JOIN rental r      ON p.rental_id    = r.rental_id
JOIN inventory i   ON r.inventory_id = i.inventory_id
JOIN film_actor fa ON i.film_id      = fa.film_id
JOIN actor a       ON fa.actor_id    = a.actor_id
GROUP BY a.actor_id, a.first_name, a.last_name
ORDER BY total_revenue DESC
LIMIT 5;
```

**핵심 사고법:** JOIN 쓰기 전에 "어떤 테이블이 다리 역할인가?" 를 먼저 그려보자.

```
payment → rental → inventory → film_actor → actor
  결제  →  임대  →    재고   →  영화-배우  →  배우
```

---

## 패턴 C — 멘토링 매칭 문제 (Self JOIN 응용)

> "같은 테이블에서 멘티와 멘토를 동시에 뽑아 매칭하라." → 이 문제는 **Self JOIN** 이다. → [[SQL_Self_Join]] 참고

**문제 조건:**

- 멘티: 2021-12-31 기준 3개월 이내 입사 (`> 2021-09-30`)
- 멘토: 2021-12-31 기준 재직 2년 이상 (`<= 2019-12-31`)
- 서로 다른 부서끼리만 매칭
- 멘토가 없어도 멘티는 전부 출력 (NULL 허용)
- 정렬: 멘티 ID 오름차순 → 멘토 ID 오름차순

**풀이 사고 흐름:**

```
1. 기준이 되는 쪽은? → 멘티 (매칭 멘토가 없어도 출력해야 함)
2. 어떤 JOIN?        → LEFT JOIN (멘티가 주인공, 멘토는 없으면 NULL)
3. 멘티 필터는?      → WHERE 절 (기준 테이블 e1 자체를 거름)
4. 멘토 필터는?      → ON 절   (조인 대상 e2를 거르되 e1은 보존)
5. 다른 부서 조건은? → ON 절   (조인 조건이므로)
```

```sql
SELECT
    e1.employee_id   AS mentee_id,
    e1.name          AS mentee_name,
    e2.employee_id   AS mentor_id,
    e2.name          AS mentor_name
FROM employees e1                    -- e1 = 멘티 후보 (기준 테이블)
LEFT JOIN employees e2               -- e2 = 멘토 후보 (같은 테이블 Self JOIN)
    ON  e2.hire_date  <= '2019-12-31'  -- ✅ ON: 멘토 조건 (조인 대상 필터)
    AND e1.department_id <> e2.department_id  -- ✅ ON: 다른 부서 조건
WHERE e1.hire_date > '2021-09-30'    -- ✅ WHERE: 멘티 조건 (기준 테이블 필터)
ORDER BY mentee_id ASC, mentor_id ASC;
```

**왜 이 구조인가?**

```
WHERE e1.hire_date > '2021-09-30'
→ e1(멘티) 자체를 필터링 → WHERE 절이 맞다
→ 이 조건은 e1 행을 지워도 괜찮음 (멘티 자격 없는 사람은 아예 불필요)

ON e2.hire_date <= '2019-12-31'
→ e2(멘토)를 필터링하되 e1은 보존 → ON 절이 맞다
→ 멘토 조건을 WHERE에 두면 멘토 없는 멘티까지 제거됨 → 문제 요구사항 위반!
```

> ✍️ **SQLD·면접용 한 줄 회고** `LEFT JOIN` 에서 **기준 테이블 필터는 WHERE, 조인 대상 필터는 ON 절** 에 둔다.

---

## 패턴 D — OR 조인 대신 UNION ALL ⭐️ 성능 주의

> "두 컬럼 중 하나라도 일치하면 조인" → OR 조건 쓰면 성능 급격히 저하

```text
문제 상황:
  대여 기록 테이블에 rent_station_id, return_station_id 두 컬럼이 있고
  두 컬럼 중 하나라도 특정 station 과 매칭되면 조인하고 싶을 때

  직관적으로 OR 로 쓰고 싶어짐
```

```sql
-- ❌ OR 조건 JOIN — PostgreSQL 에서 성능 저하 (인덱스 사용 불가)
SELECT r.*, s.station_name
FROM rental r
JOIN station s
    ON r.rent_station_id   = s.id
    OR r.return_station_id = s.id;
```

```text
왜 느린가:
  OR 조건은 인덱스를 타지 못하고 Full Scan 으로 처리됨
  데이터가 많을수록 성능이 기하급수적으로 나빠짐
  PostgreSQL 은 OR JOIN 을 효율적으로 최적화하지 못함
```

```sql
-- ✅ UNION ALL 로 대체 — OR 없이 같은 결과, 성능 정상
SELECT r.*, s.station_name, 'rent' AS type
FROM rental r
JOIN station s ON r.rent_station_id = s.id

UNION ALL

SELECT r.*, s.station_name, 'return' AS type
FROM rental r
JOIN station s ON r.return_station_id = s.id;
```

```text
UNION ALL 동작 원리:
  ① 출발지(rent_station_id) 기준으로 조인한 결과
  ② 도착지(return_station_id) 기준으로 조인한 결과
  두 결과를 위아래로 쌓음

  각각의 JOIN 은 단일 컬럼 조건 → 인덱스 정상 사용 ✅
  OR 없이도 두 케이스를 전부 커버
```

>[[SQL_UNION]] 참고 
---

---

# SQLD 빈출 — 최소 JOIN 조건 개수

> [!tip] **N개의 테이블을 조인하려면 최소 N-1개의 JOIN 조건이 필요하다.**

|테이블 수|최소 JOIN 조건 수|
|---|---|
|2개|1개|
|3개|2개|
|4개|3개|
|N개|**N-1개**|

```
A ─── B ─── C ─── D
 (1)   (2)   (3)
섬 4개 → 다리 3개(N-1)로 전부 연결 가능
```

> ⚠️ JOIN 조건이 N-1개보다 **적으면** → 연결 안 된 테이블이 **CROSS JOIN** 되어 데이터가 기하급수적으로 뻥튀기된다.

---

---

# 초보자가 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|LEFT JOIN 했는데 행 수가 늘어남|오른쪽 테이블에 N개 매칭 → 곱하기 발생|1:N 관계 먼저 파악|
|LEFT JOIN 했는데 INNER JOIN 결과가 나옴|INNER 쪽 조건을 WHERE 절에 둠|조인 대상 조건은 ON 절로 이동|
|"없는 것" 을 찾는데 INNER JOIN 씀|조인 시점에 이미 데이터 제거됨|LEFT JOIN + `WHERE B.id IS NULL`|
|ON 절에 조건 하나만 써야 한다고 생각|오해|`ON A.id = B.id AND A.date > '2024-01-01'` 처럼 AND로 추가 가능|
|`ON A.id = B.a OR A.id = B.b` 성능 저하|OR 조건 → 인덱스 사용 불가, Full Scan|UNION ALL 로 분리해서 각각 조인|
|NULL 처리 안 함|LEFT JOIN 결과의 NULL을 계산에 사용|`COALESCE(col, 0)` 로 처리|
|최댓값이 여러 개인데 1개만 나올 거라 생각|공동 1등은 전부 출력됨|딱 1개만 원하면 `ROW_NUMBER()` 사용|

---
