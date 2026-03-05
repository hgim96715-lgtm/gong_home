---
aliases:
  - SQL WHERE
  - SQL 필터링
  - 조건문
  - IN
  - LIKE
  - IS NULL
  - ESCAPE
  - 드모르간
  - ILIKE
tags:
  - SQL
related:
  - "[[SQL_SELECT_FROM]]"
  - "[[HAVING_vs_WHERE]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Understanding_NULL]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
---
---

# SQL 필터링 — WHERE 절

## 개념 한 줄 요약

> **"엑셀의 필터 기능과 완전히 같다. 원하는 조건에 맞는 행(Row)만 쏙 뽑아낸다."**

---

## 왜 필요한가?

|구분|설명|
|---|---|
|문제 😱|`SELECT * FROM Orders` 하면 1억 건이 쏟아져 서버가 다운될 수 있다|
|해결 ✅|"서울 사는 사람", "어제 주문 건" 처럼 조건을 걸어 필요한 데이터만 가져온다|

---

## 조건식의 좌우 대칭 법칙

> 컬럼명이 반드시 왼쪽에 있어야 하는 건 아니다. `{text}=` `<` `>` 기준으로 양옆을 바꿔도 SQL 은 똑같이 동작한다.

```sql
-- 아래 두 쿼리는 100% 동일
SELECT * FROM users WHERE age = 20;
SELECT * FROM users WHERE 20 = age;
```

---

---

# ① 비교 연산자

|구분|연산자|의미|예시|
|---|---|---|---|
|긍정|`=`|같다|`age = 20`|
||`>` `<`|크다, 작다|`price > 1000`|
||`>=` `<=`|크거나/작거나 같다|`score <= 50`|
|부정|`<>`|다르다 **(표준 SQL) ⭐️**|`city <> 'Seoul'`|
||`!=`|다르다 (대부분 지원)|`city != 'Seoul'`|
||`^=`|다르다 **(Oracle 전용)**|`city ^= 'Seoul'`|
||`NOT 컬럼 =`|같지 않다|`NOT city = 'Seoul'`|
||`NOT 컬럼 >`|크지 않다 (`<=` 와 동일)|`NOT price > 1000`|

---

---

# ② 논리 연산자 — AND · OR · NOT

|연산자|의미|집합 개념|
|---|---|---|
|`AND`|두 조건이 **모두** 참|교집합|
|`OR`|둘 중 **하나라도** 참|합집합|
|`NOT`|조건의 반대|여집합|

## 연산자 우선순위 ⭐️

```
1순위: ( )
2순위: 비교 연산자 (= > < >= <=)
3순위: NOT
4순위: AND
5순위: OR
```

> **AND 가 OR 보다 먼저 계산된다!** `A OR B AND C` = `A OR (B AND C)` → 섞어 쓸 땐 반드시 괄호를 치자.

## 드 모르간의 법칙

> NOT 이 괄호 안으로 들어가면 **AND ↔ OR** 로 바뀐다.

```
NOT (A OR B)  →  NOT A AND NOT B
NOT (A AND B) →  NOT A OR  NOT B
```

```sql
-- 불량 데이터 청소: 종(species) 또는 몸무게(body_mass_g) 하나라도 NULL 이면 제거

-- 방법 1 (드 모르간): 나쁜 것들을 OR 로 묶고 NOT
WHERE NOT (species IS NULL OR body_mass_g IS NULL)

-- 방법 2 (동일 결과): 온전한 것만 AND 로 남기기
WHERE species IS NOT NULL AND body_mass_g IS NOT NULL
```

---

---

# ③ BETWEEN — 범위 조건

> **양 끝값을 모두 포함(Inclusive) 한다. `>= A AND <= B` 와 동일.**

```sql
WHERE age BETWEEN 10 AND 20       -- 10 이상 20 이하 (10, 20 포함)
WHERE age NOT BETWEEN 10 AND 20   -- 10~20 제외한 나머지
```

## ⚠️ 날짜 사용 시 주의

```sql
-- ❌ 위험: '2026-03-05' 는 00시 00분 00초까지만 포함
WHERE created_at BETWEEN '2026-03-04' AND '2026-03-05'
-- → 3월 5일 오후 데이터 누락!

-- ✅ 안전: 부등호 사용 권장
WHERE created_at >= '2026-03-04' AND created_at < '2026-03-06'
```

---

## ⭐️ 값 BETWEEN 컬럼1 AND 컬럼2 — 가장 많이 착각하는 패턴

> **BETWEEN 왼쪽에 컬럼이 아니라 값(상수)이 올 수 있다.**

```sql
-- 일반적인 패턴 (컬럼이 주어)
WHERE COL1 BETWEEN 100 AND 300
-- = COL1 >= 100 AND COL1 <= 300
-- "COL1 의 값이 100~300 사이인가?"

-- ❌ 착각하기 쉬운 패턴 (값이 주어)
WHERE 200 BETWEEN COL1 AND COL2
-- "200 이 COL1 과 COL2 사이에 있는가?"
-- = COL1 <= 200 AND 200 <= COL2
-- = COL1 <= 200 AND COL2 >= 200
```

**예제 테이블:**

|ROW|COL1|COL2|
|---|---|---|
|1|100|300|
|2|150|180|
|3|201|400|
|4|50|199|

```sql
SELECT * FROM T WHERE 200 BETWEEN COL1 AND COL2;
```

|ROW|COL1|COL2|COL1 <= 200?|COL2 >= 200?|통과?|
|---|---|---|:-:|:-:|:-:|
|1|100|300|✅|✅|✅|
|2|150|180|✅|❌ (180 < 200)|❌|
|3|201|400|❌ (201 > 200)|✅|❌|
|4|50|199|✅|❌ (199 < 200)|❌|

> **결과: 1행만 통과**
> 
> 착각 포인트: "COL1=200 이나 COL2=200 인 행을 찾는 것" 이 아니다. "200 이라는 값이 COL1~COL2 범위 안에 포함되는가?" 를 묻는 것이다.

```
암기 공식:
A BETWEEN B AND C
= B <= A AND A <= C
= "A 가 B 와 C 사이에 있는가?"

주어(기준값)는 항상 BETWEEN 왼쪽이다.
```

---

---

# ④ IN — 집합 조건

> **OR 를 여러 번 쓰는 노가다를 줄여준다.**

```sql
WHERE city IN ('Seoul', 'Busan', 'Daegu')
-- = WHERE city = 'Seoul' OR city = 'Busan' OR city = 'Daegu'

WHERE city NOT IN ('Seoul', 'Busan')
-- Seoul, Busan 이 모두 아닌 것
```

## ⚠️ NOT IN + NULL 치명적 함정

```sql
-- NOT IN 목록 안에 NULL 이 하나라도 있으면 → 결과 0건!
WHERE id NOT IN (1, 2, NULL)
-- NULL 과의 비교는 항상 UNKNOWN → 모든 행 탈락

-- 해결: NULL 명시적 제거
WHERE id NOT IN (SELECT id FROM T WHERE id IS NOT NULL)
```

> → [[SQL_SubQuery#NULL 과 IN 의 관계]] 참고

---

---

# ⑤ LIKE / ILIKE — 패턴 매칭

|연산자|대소문자 구분|DB|
|---|:-:|---|
|`LIKE`|✅ 구분 함|모든 DB|
|`ILIKE`|❌ 구분 안 함|PostgreSQL 전용|
|`NOT LIKE` / `NOT ILIKE`|—|패턴 불포함|

## 와일드카드

|기호|의미|예시|
|---|---|---|
|`%`|모든 문자 (0글자 이상)|`김%` → 김, 김수, 김철수...|
|`_`|딱 한 글자|`김_수` → 김철수 ✅, 김영희수 ❌|

```sql
WHERE name LIKE '김%'        -- 김으로 시작
WHERE name LIKE '%수'        -- 수로 끝남
WHERE name LIKE '%길%'       -- 길이 포함
WHERE name LIKE '김_수'      -- 김X수 (가운데 1글자)
WHERE name NOT LIKE '%테스트%' -- 테스트 미포함
```

## ESCAPE — 진짜 % · _ 검색

>`ESCAPE '지정문자'` → 지정문자 바로 뒤에 오는 1글자를 와일드카드가 아닌 일반 문자로 처리

```sql
-- '%50%' 라는 문자열 자체를 찾고 싶을 때
WHERE review LIKE '%50\%%' ESCAPE '\'
--                    ↑↑
--                    \ %
--                    │ └ 진짜 % 문자 (와일드카드 아님)
--                    └ ESCAPE 지정문자 → 바로 뒤를 일반 문자로 처리

-- _ 가 포함된 경로 찾기 (PostgreSQL)
WHERE page_location ILIKE '%\_%' ESCAPE '\'
--                           \_ = 진짜 _ 문자
```

```sql
-- \ 말고 다른 문자로도 지정 가능 (단, 딱 1글자)

WHERE review LIKE '%50!%%' ESCAPE '!'   -- ! 를 이스케이프 문자로 지정
WHERE review LIKE '%50@%%' ESCAPE '@'   -- @ 를 이스케이프 문자로 지정
```

## 여러 패턴 한 번에 — ILIKE ANY (PostgreSQL 특화)

```sql
-- ❌ OR 반복 (지저분)
WHERE email ILIKE '%gmail%' OR email ILIKE '%naver%'

-- ✅ ILIKE ANY (깔끔)
WHERE email ILIKE ANY (ARRAY['%gmail%', '%naver%'])
```

## ⚠️ 비율 계산 시 LIKE 금지

```sql
-- ❌ 위험: 기프트카드 아닌 데이터가 전부 날아감 (분모 실종)
SELECT COUNT(*) FROM payments
WHERE credit ILIKE '%gift%'

-- ✅ 안전: CASE WHEN 으로 조건부 집계
SELECT
    SUM(CASE WHEN credit ILIKE '%gift%' THEN 1 ELSE 0 END) AS gift_count,
    COUNT(*) AS total_count
FROM payments
```

> → [[SQL_CASE_WHEN]] 참고

---

---

# ⑥ IS NULL — 결측치 확인

> **NULL 은 "알 수 없음" 상태. `{text}=` `<>` 로 비교가 절대 불가능하다.**

```sql
-- ❌ 절대 안 됨
WHERE age = NULL      -- 항상 FALSE
WHERE age <> NULL     -- 항상 FALSE

-- ✅ 무조건 IS NULL / IS NOT NULL
WHERE age IS NULL
WHERE age IS NOT NULL
```

---

---

# ⑦ 고급 패턴 매칭 — 정규표현식 (PostgreSQL)

|연산자|대소문자 구분|의미|
|---|:-:|---|
|`~`|✅|매칭됨|
|`~*`|❌|매칭됨 (대소문자 무시)|
|`!~`|✅|매칭 안 됨|
|`!~*`|❌|매칭 안 됨 (대소문자 무시)|

```sql
-- 'fire' 또는 'water' 포함 (대소문자 무시)
WHERE description ~* 'fire|water'

-- 이름이 A 또는 B 로 시작
WHERE name ~* '^(A|B)'
```

> **IN 안에 와일드카드(%) 는 불가능하다.** IN 은 완전 일치(Exact Match) 만 비교한다.

```sql
 -- ❌ 아무것도 안 나옴 — % 자체를 문자로 찾음
 WHERE email IN ('%@gmail.com', '%@naver.com')
```

---

---

# ⑧ WHERE 절의 제한 사항

## 별칭(Alias) 사용 불가

```
실행 순서: FROM → WHERE → SELECT
WHERE 는 SELECT 보다 먼저 실행 → 아직 별칭이 없음
```

```sql
-- ❌ 에러: 'total' 이 뭔지 모름
SELECT price * quantity AS total FROM orders WHERE total > 1000;

-- ✅ 원본 컬럼으로 직접 계산
SELECT price * quantity AS total FROM orders WHERE price * quantity > 1000;
```

## 집계 함수 사용 불가

```sql
-- ❌ 에러: WHERE 는 개별 행 검사 → 그룹 집계 모름
SELECT dept, AVG(salary) FROM emp WHERE AVG(salary) > 5000;

-- ✅ 집계 조건은 HAVING 사용
SELECT dept, AVG(salary) FROM emp GROUP BY dept HAVING AVG(salary) > 5000;
```

---

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|NULL 찾을 때 `=` 사용|NULL 은 비교 불가|`IS NULL` 사용|
|AND · OR 혼용 시 괄호 미사용|AND 가 OR 보다 우선|괄호 `()` 필수|
|문자열에 따옴표 없음|숫자와 문자 구분|문자는 `'Text'`|
|BETWEEN 끝값 제외 착각|양 끝값 모두 포함|`>=` `<=` 와 동일|
|`값 BETWEEN COL1 AND COL2` 오해|주어가 값(상수)|`COL1 <= 값 AND 값 <= COL2`|
|NOT IN 안에 NULL 포함|전체 결과 0건|IS NOT NULL 로 제거|
|WHERE 에 별칭 사용|실행 순서 문제|원본 컬럼으로 계산|
|WHERE 에 집계 함수 사용|개별 행 처리 단계|HAVING 사용|

---
