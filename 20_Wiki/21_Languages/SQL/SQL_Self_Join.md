---
aliases:
  - 셀프 조인
  - Self Jon
  - 자기참조 조인
tags:
  - SQL
related:
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_Standard_JOIN]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Execution_Order]]"
---
# SQL_Self_Join — 셀프 조인

## 한 줄 요약

```
같은 테이블에 별칭을 두 개 붙여
마치 서로 다른 두 테이블인 것처럼 자기 자신과 JOIN
```

---

---

# ① Self Join 이 왜 필요한가

```
일반 JOIN:  서로 다른 두 테이블 연결
Self JOIN:  같은 테이블 안에서 행끼리 비교

예시:
  사원 테이블에서 "각 사원의 매니저 이름" 을 찾으려면
  → 사원 테이블과 사원 테이블을 연결해야 함
  → 같은 테이블을 두 번 사용 = Self JOIN
```

## 별칭(Alias) 필수 ⚠️

```sql
-- ❌ 에러 — DB 가 어느 테이블 참조인지 구분 불가
SELECT 사원명 FROM 사원 JOIN 사원 ON ...

-- ✅ 별칭으로 구분
SELECT e1.사원명 FROM 사원 e1 JOIN 사원 e2 ON ...
```

---

---

# ② 패턴 1 — 등가 조인 (계층 구조)

## 사원-매니저 관계

```
사원 테이블 하나에
  사원번호 / 사원명 / 매니저번호
가 모두 들어있음

"이순신의 매니저는?" 을 알려면
  이순신 행의 매니저번호 = 매니저 행의 사원번호
  → 같은 테이블을 사원(e1) / 매니저(e2) 로 나눠서 JOIN
```

```sql
SELECT
    e1.사원명  AS 사원,
    e2.사원명  AS 매니저
FROM 사원 e1
LEFT JOIN 사원 e2
    ON e1.매니저번호 = e2.사원번호   -- e1의 매니저번호 = e2의 사원번호
```

```
LEFT JOIN 이유:
  홍길동은 매니저가 없는 최상위 사원
  INNER JOIN 이면 홍길동 행이 사라짐
  LEFT JOIN 이면 매니저 NULL 로 포함
```

|사원|매니저|
|---|---|
|홍길동|NULL (최상위)|
|이순신|홍길동|
|이민정|홍길동|

---

---

# ③ 패턴 2 — 범위 조건 조인 (누적합)

## 원본 데이터

|일자|매출액|
|---|---|
|11.01|1000|
|11.02|1000|
|11.03|1000|

## 쿼리

```sql
SELECT
    A.일자,
    SUM(B.매출액) AS 누적매출액
FROM 일자별매출 A
JOIN 일자별매출 B
    ON A.일자 >= B.일자   -- A 날짜 기준, 같거나 과거인 B 전부 매칭
GROUP BY A.일자
ORDER BY A.일자;
```

## 내부 동작 — A.일자 = 11.03 일 때

```
1단계: 같은 테이블을 A, B 두 개로 복제

  A.일자  A.매출  |  B.일자  B.매출
  11.03   1000   |  11.01   1000
  11.03   1000   |  11.02   1000
  11.03   1000   |  11.03   1000

2단계: ON A.일자 >= B.일자 조건 적용
  A = 11.03 이면 B = 11.01 / 11.02 / 11.03 모두 조건 만족

  A.일자  │  B.일자  B.매출
  11.03   │  11.01   1000    ← 11.03 >= 11.01 ✅
  11.03   │  11.02   1000    ← 11.03 >= 11.02 ✅
  11.03   │  11.03   1000    ← 11.03 >= 11.03 ✅

3단계: GROUP BY A.일자 + SUM(B.매출액)
  11.03 → 1000 + 1000 + 1000 = 3000
```

## 부등호 방향에 따른 차이

|조건|동작|결과|
|---|---|---|
|`A.일자 >= B.일자`|과거 데이터 합산|날짜 갈수록 증가|
|`A.일자 <= B.일자`|미래 데이터 합산|날짜 갈수록 감소|

---

---

# ④ 패턴 3 — 조건부 매칭 (멘토-멘티)

## 문제 조건

```
같은 employees 테이블에서
  멘티: 3개월 이내 신입 (hire_date > 2021-09-30)
  멘토: 재직 2년 이상   (hire_date <= 2019-12-31)
  조건: 서로 다른 부서 / 멘토 없어도 멘티는 전부 출력
```

## 쿼리

```sql
SELECT
    e1.employee_id  AS mentee_id,
    e1.name         AS mentee_name,
    e2.employee_id  AS mentor_id,
    e2.name         AS mentor_name
FROM employees e1                              -- e1 = 멘티 (기준)
LEFT JOIN employees e2                         -- e2 = 멘토 (같은 테이블)
    ON  e2.hire_date <= '2019-12-31'           -- ON: 멘토 조건
    AND e1.department_id != e2.department_id   -- ON: 다른 부서
WHERE e1.hire_date > '2021-09-30'              -- WHERE: 멘티 조건
ORDER BY mentee_id, mentor_id;
```

```
ON 절 vs WHERE 절 구분:
  WHERE e1.hire_date > ...  → 멘티 자격 없는 행 자체 제거
  ON e2.hire_date <= ...    → 멘토만 필터 (e1은 보존)
  ON 조건을 WHERE 에 두면 멘토 없는 멘티까지 사라짐 ❌
```

---

---

# ⑤ 패턴 4 — 3자 관계 (삼각형 친구 찾기) ⭐️

## 문제

```
edges 테이블: 친구 관계 (user_a_id, user_b_id)
"세 명이 모두 서로 친구인 관계(삼각형)" 를 찾아라

     A
    / \
   B - C
A-B 친구, B-C 친구, A-C 친구 → 삼각형
```

## 핵심 아이디어 — 먼저 양방향으로 만들기

```
원본 edges:
  (1,2) → 1과 2는 친구
  (2,3) → 2와 3은 친구
  (1,3) → 1과 3은 친구

문제: 1→2 는 있는데 2→1 이 없으면 JOIN 연결 안 됨
해결: UNION ALL 로 반대 방향 추가

all_edges 결과:
  u1  u2
  1   2   ← 원본
  2   3   ← 원본
  1   3   ← 원본
  2   1   ← 추가 (뒤집기)
  3   2   ← 추가 (뒤집기)
  3   1   ← 추가 (뒤집기)
```

## 삼각형 찾는 원리

```
e1 테이블: A→B 관계
e2 테이블: B→C 관계 (e1.u2 = e2.u1 로 연결)
e3 테이블: A→C 관계 (삼각형 완성 확인)

e1: 1→2  (A=1, B=2)
e2: 2→3  (B=2, C=3)   ← e1.u2(=2) = e2.u1(=2) 로 연결
e3: 1→3  (A→C 존재?)  ← e1.u1(=1)=e3.u1, e2.u2(=3)=e3.u2 확인
→ (1,3) 이 all_edges 에 있음 → 삼각형 완성!
```

## 쿼리

```sql
WITH all_edges AS (
    SELECT user_a_id AS u1, user_b_id AS u2 FROM edges
    UNION ALL
    SELECT user_b_id AS u1, user_a_id AS u2 FROM edges
    -- A-B 있으면 B-A 도 추가 (양방향)
)
SELECT
    e1.u1 AS user_a_id,
    e1.u2 AS user_b_id,
    e2.u2 AS user_c_id
FROM all_edges e1
JOIN all_edges e2 ON e1.u2 = e2.u1          -- A→B, B→C 연결
JOIN all_edges e3 ON e1.u1 = e3.u1          -- A→C 존재 확인
                 AND e2.u2 = e3.u2
WHERE e1.u1 < e1.u2                         -- 중복 제거
  AND e1.u2 < e2.u2
```

## 중복 제거 조건 이해

```
삼각형 (1,2,3) 이 있으면
양방향 테이블에서 6가지 순서 조합이 나옴:
  (1,2,3) (1,3,2) (2,1,3) (2,3,1) (3,1,2) (3,2,1)

WHERE e1.u1 < e1.u2 AND e1.u2 < e2.u2
  → u1 < u2 < u3 인 오름차순 조합만 남김
  → (1,2,3) 딱 하나만 결과에 나옴
```

## 실무 활용

```
SNS:    "A, B, C 가 모두 아는 사람" 친구 추천
커머스: "이 상품을 함께 구매한 고객 그룹"
추천:   협업 필터링 기초
조직:   3자 연결 관계 / 연결망 분석
```

---

---

# Self Join vs 윈도우 함수

```
실무에서는 윈도우 함수가 훨씬 빠름
Self Join 은 테이블을 N번 스캔 → 데이터 폭증
윈도우 함수는 테이블 1회 스캔
SQLD 시험에는 Self Join 출제
```

```sql
-- Self Join 누적합
SELECT A.일자, SUM(B.매출액) AS 누적
FROM 일자별매출 A
JOIN 일자별매출 B ON A.일자 >= B.일자
GROUP BY A.일자;

-- 윈도우 함수 (실무 권장)
SELECT 일자, SUM(매출액) OVER (ORDER BY 일자) AS 누적
FROM 일자별매출;
```

|항목|Self Join|윈도우 함수|
|---|---|---|
|성능|느림 (데이터 폭증)|빠름 (1회 스캔)|
|코드|길다|짧다|
|SQLD|⭐️ 빈출|실무 우선|