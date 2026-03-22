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


# SQL_Standard_JOIN — JOIN 완전 정리

## 한 줄 요약

```
공통 키(Key) 를 기준으로 두 테이블을 하나로 합치는 연산
데이터가 분리된 여러 테이블을 연결해서 분석
```

---

---

# ① JOIN 의 핵심 — ON 은 조건, 행 전체가 붙음

```
가장 많이 하는 오해:
  "ON 절에 쓴 컬럼만 가져온다"

실제:
  ON 은 "어떤 행을 붙일지" 찾는 조건
  조건에 맞는 행 전체가 붙어옴
```

```sql
SELECT *
FROM orders o
JOIN users u ON o.user_id = u.id

-- ON 조건: u.id 만 비교
-- 하지만 JOIN 되면 u.name / u.email / u.age 전부 딸려옴
-- ON 은 매칭 조건 / 붙는 건 행 전체
```

```
비유:
  "2번 서랍 열어줘" (ON 조건)
  → 서랍 번호(u.id) 만 나오는 게 아니라
  → 서랍 안 내용물 전부 (u.name, u.email ...) 나옴
```

---

---

# ② ON 절 vs WHERE 절 ⭐️

```
가장 많이 헷갈리는 부분
```

|구분|실행 시점|하는 일|
|---|---|---|
|`ON` 절|테이블 엮는 순간|조건 불일치 → 오른쪽 NULL (왼쪽 보존)|
|`WHERE` 절|엮은 결과 전체에서|조건 불일치 → 행 자체 삭제|

## 원칙

```
기준 테이블(FROM) 의 필터  →  WHERE 절
조인 대상 테이블의 필터    →  ON 절
```

## 코드 비교

```sql
-- ✅ ON 절에 조인 대상 조건
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON  A.id = B.customer_id
    AND B.status = 'DONE'    -- B 조건 → ON 절 (A 는 보존)

-- ❌ WHERE 절에 조인 대상 조건
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON  A.id = B.customer_id
WHERE B.status = 'DONE'      -- B 가 NULL 인 행 전부 삭제
                              -- → 사실상 INNER JOIN 과 동일
```

## 판단 기준

```
조건을 걸 때마다:

Q. 이 조건이 FROM 테이블(기준) 의 행을 지워도 괜찮아?
   YES → WHERE 절  (기준 테이블 자체 필터링)
   NO  → ON 절     (조인 대상 필터링, 기준 보존)
```

---

---

# ③ JOIN 종류

## INNER JOIN — 교집합 ⭐️ 가장 많이 씀

```
양쪽 모두에 데이터 있는 행만 남김
짝이 없으면 버려짐 → 결과 행 수 줄어들 수 있음
```

```sql
SELECT A.customer_name, B.order_date
FROM Customer A
INNER JOIN Orders B ON A.id = B.customer_id
-- 주문 내역 있는 고객만 / 주문 안 한 고객 사라짐
```

## LEFT JOIN — 왼쪽 기준 ⭐️ 실무에서 가장 많이 씀

```
왼쪽 테이블 무조건 전부 살림
오른쪽 없으면 NULL 로 채움
```

```sql
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B ON A.id = B.customer_id
-- 주문 안 한 고객도 NULL 상태로 포함
```

```sql
-- "없는 것" 찾기 패턴 — LEFT JOIN + IS NULL
SELECT a.author_name
FROM Authors a
LEFT JOIN Books b ON a.id = b.author_id
WHERE b.id IS NULL     -- 책이 없는 작가만
```

```
❌ INNER JOIN 으로 "없는 것" 찾기:
  조인 시점에 이미 짝 없는 행 제거됨
  WHERE COUNT = 0 해봤자 이미 날아간 데이터 못 살림
```

## RIGHT JOIN — 오른쪽 기준

```
LEFT JOIN 과 방향만 반대
실무에서는 테이블 위치 바꿔서 LEFT JOIN 으로 통일 (가독성)
```

## FULL OUTER JOIN — 합집합

```
양쪽 모두 전부 살림 / 없으면 NULL
MySQL 미지원 → LEFT JOIN + RIGHT JOIN UNION 으로 구현
실무 사용 빈도 낮음
```

## CROSS JOIN — 카테시안 곱

```
조건 없이 모든 경우의 수 곱하기
행 수 = A테이블 × B테이블
컬럼 수 = A컬럼 + B컬럼

1000건 × 1000건 = 100만 건 순식간에 생성
실무: 더미 데이터 생성 / 조합 테이블 만들 때만
```

```sql
SELECT BOY_NAME, GIRL_NAME FROM GIRL CROSS JOIN BOY;
SELECT BOY_NAME, GIRL_NAME FROM GIRL, BOY;  -- 동일
```

---

---

# ④ NATURAL JOIN / USING

## NATURAL JOIN — 이름 같은 컬럼 자동 조인

```
두 테이블에서 이름이 같은 컬럼을 자동으로 전부 찾아 조인
이름 같은 모든 컬럼 값이 100% 동일한 행만 결과에 나옴
```

```sql
SELECT * FROM 사원 NATURAL JOIN 부서;
-- 이름이 같은 컬럼 (dept_id, status 등) 전부 자동 조인 기준
```

```
⚠️ 실무에서 절대 사용 금지:
  나중에 누군가 같은 이름 컬럼 추가하면
  기존 쿼리가 소리소문없이 망가짐
```

## USING — 내가 지정한 컬럼만 조인

```sql
-- ON 방식
JOIN Orders B ON A.id = B.id

-- USING 축약
JOIN Orders B USING(id)   -- A.id = B.id 와 동일
```

## 공통 주의 — 테이블명 붙이면 에러

```sql
-- NATURAL JOIN / USING 으로 조인한 컬럼은
-- SELECT 에서 테이블명 / 별칭 붙이면 에러

SELECT A.dept_id  -- ❌ 에러
FROM 사원 A NATURAL JOIN 부서 B;

SELECT dept_id    -- ✅ 테이블명 없이
FROM 사원 A NATURAL JOIN 부서 B;
```

```
이유:
  DB 가 두 테이블의 해당 컬럼을 하나로 합쳐서 처리
  어느 테이블 소속인지 구분이 없어짐
```

|구분|조인 기준|별칭 사용|안전성|
|---|---|---|---|
|NATURAL JOIN|이름 같은 컬럼 전부 자동|❌ 조인 컬럼 불가|❌ 위험|
|USING(id)|지정한 컬럼만|❌ 조인 컬럼 불가|✅|
|ON A.id = B.id|명시적 지정|✅ 전부 가능|✅ 가장 안전|

---

---

# ⑤ 행 수 뻥튀기 — 가장 많이 터지는 사고

```
JOIN 결과 행 수 = 왼쪽 매칭 건수 × 오른쪽 매칭 건수
```

```
예시: 3번 회원이 주문 2건

왼쪽 (회원): 1, 3, 5
오른쪽 (주문): 주문A(고객1), 주문B(고객3), 주문C(고객3)

결과:
  고객1 → 주문A            1줄
  고객3 → 주문B, 주문C     2줄  ← 뻥튀기!
  고객5 → NULL             1줄
  총 4줄  (예상 3줄이 아님)
```

```
항상 JOIN 전에 1:N 관계 파악 필수
N:M 관계면 결과가 더 크게 터짐
```

---

---

# ⑥ 고급 패턴

## 다중 조건 JOIN (Groupwise Max)

```sql
-- 요일별 가장 비싼 결제 건
SELECT t.*
FROM tips t
JOIN (
    SELECT day, MAX(total_bill) AS max_bill
    FROM tips
    GROUP BY day
) m
  ON  t.day = m.day
  AND t.total_bill = m.max_bill
-- ON 절에 AND 로 조건 여러 개 가능
```

## 체인 JOIN (다중 테이블)

```sql
-- 연결고리만 있으면 몇 개든 이어붙임
SELECT a.first_name, SUM(p.amount)
FROM payment p
JOIN rental r      ON p.rental_id    = r.rental_id
JOIN inventory i   ON r.inventory_id = i.inventory_id
JOIN film_actor fa ON i.film_id      = fa.film_id
JOIN actor a       ON fa.actor_id    = a.actor_id
GROUP BY a.actor_id
ORDER BY 2 DESC LIMIT 5;
```

```
JOIN 짜기 전에 연결 흐름 먼저 그리기:
  payment → rental → inventory → film_actor → actor
```

## OR 조인 → UNION ALL 로 교체 ⭐️

```
OR 조인은 인덱스 사용 불가 → Full Scan → 성능 저하

두 컬럼 중 하나라도 일치하면 조인:
```

```sql
-- ❌ OR 조인 — 인덱스 못 탐
JOIN station s
    ON r.rent_station_id   = s.id
    OR r.return_station_id = s.id

-- ✅ UNION ALL — 각각 단일 조건으로 분리
SELECT r.*, s.station_name, 'rent' AS type
FROM rental r
JOIN station s ON r.rent_station_id = s.id

UNION ALL

SELECT r.*, s.station_name, 'return' AS type
FROM rental r
JOIN station s ON r.return_station_id = s.id
```

---

---

# ⑦ Oracle (+) → ANSI 변환 — SQLD 빈출

```sql
-- Oracle (+) 방식
SELECT A.customer_name, B.order_date
FROM Customer A, Orders B
WHERE A.id = B.customer_id(+)
  AND B.status(+) = 'DONE'   -- INNER 쪽(B) 조건에 (+) 붙임

-- ✅ ANSI 변환 — INNER 쪽 조건 → ON 절로
SELECT A.customer_name, B.order_date
FROM Customer A
LEFT JOIN Orders B
    ON  A.id = B.customer_id
    AND B.status = 'DONE'    -- ON 절에 위치

-- ❌ 잘못된 변환
FROM Customer A
LEFT JOIN Orders B ON A.id = B.customer_id
WHERE B.status = 'DONE'      -- WHERE 절 → INNER JOIN 이 됨
```

---

---

# SQLD 빈출 — 최소 JOIN 조건 개수

```
N개의 테이블 조인 → 최소 N-1개의 JOIN 조건 필요

A ─── B ─── C ─── D
 (1)   (2)   (3)
섬 4개 → 다리 3개(N-1) 로 전부 연결

JOIN 조건이 N-1 개보다 적으면:
  연결 안 된 테이블이 CROSS JOIN 됨
  → 데이터 기하급수적 뻥튀기
```

|테이블 수|최소 JOIN 조건|
|---|---|
|2개|1개|
|3개|2개|
|4개|3개|
|N개|N-1개|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|JOIN 후 행 수 늘어남|1:N 관계 → 곱하기 발생|JOIN 전 관계 파악|
|LEFT JOIN 인데 INNER 결과|조인 대상 조건을 WHERE 에|ON 절로 이동|
|"없는 것" 찾는데 INNER JOIN|조인 시점에 행 제거됨|LEFT JOIN + IS NULL|
|ON 조건 하나만 써야 한다고 생각|오해|AND 로 여러 조건 가능|
|OR JOIN 성능 저하|인덱스 사용 불가|UNION ALL 로 분리|
|ON 에 쓴 컬럼만 가져온다고 생각|오해|ON 은 조건 / 행 전체가 붙음|