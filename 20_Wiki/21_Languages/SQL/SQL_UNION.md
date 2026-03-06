---
aliases:
  - SQL 데이터 병합
  - UNION과 JOIN의 차이
  - UNION
  - UNION ALL
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_CASE_WHEN]]"
  - "[[SQL_Standard_JOIN]]"
---
# SQL_UNION 집합연산자

## 개념 한 줄 요약

두 개 이상의 독립적인 `SELECT` 쿼리 결과를 **위아래(수직)로 이어 붙이거나**, 교집합/차집합 같은 수학적 집합 계산을 해주는 연산자입니다.

---

## Why is it needed(왜 필요한가)

- **문제점:** 작년 매출 테이블(`sales_2025`)과 올해 매출 테이블(`sales_2026`)이 물리적으로 나뉘어 있는데, 한 번에 합쳐서 전체 통계를 내고 싶습니다.
- **💡 해결책:** 집합 연산자를 쓰면 A 쿼리 결과 바로 밑에 B 쿼리 결과를 테이프로 붙인 것처럼 **하나의 거대한 결과물(Result Set)** 로 만들어줍니다.

---

## Practical Context(실무맥락)

- **데이터 병합:** 용량이 너무 커서 '월별'로 쪼개놓은 로그(Log) 테이블 여러 개를 한 번에 조회할 때 씁니다.
- **제외/필터링 (차집합):** "전체 유저 중에서 블랙리스트 유저의 명단을 빼고(Minus) 조회해 줘" 같은 로직을 짤 때 아주 직관적입니다.

---

## 🔥 실전 예제 — "JOIN 했는데 데이터가 줄어요?"

### 상황: 양방향 관계 테이블

```
edges 테이블
user_a_id | user_b_id
----------+----------
    1     |    2        → 1번과 2번은 친구
    1     |    3        → 1번과 3번은 친구
    2     |    4        → 2번과 4번은 친구
```

```
문제:
각 유저의 친구 수를 구하고 싶다.

JOIN 으로 시도하면?
→ user_a_id 기준으로만 세면 user_b_id 쪽 관계가 누락
→ 1번은 2명인데 2번은 1명으로 잘못 집계됨
```

### 왜 JOIN 이 안 되는가

```
edges 테이블은 양방향 관계를 단방향으로 저장한 구조
A-B 가 친구 = B-A 도 친구인데 행은 하나만 존재

JOIN 은 수평으로 붙이는 것 → 한쪽 방향만 봄
UNION ALL 로 양방향을 수직으로 펼쳐야 양쪽을 다 셀 수 있음
```

### UNION ALL 로 해결

```sql
WITH ALLID AS (
    SELECT user_a_id AS user_id FROM edges   -- A → B 방향
    UNION ALL
    SELECT user_b_id AS user_id FROM edges   -- B → A 방향 (반대 방향도 포함)
)
-- ALLID 결과:
-- user_id
-- -------
--   1  (user_a_id 에서)
--   1  (user_a_id 에서)
--   2  (user_a_id 에서)
--   2  (user_b_id 에서)  ← 이 줄이 없으면 2번 친구 수가 누락됨
--   3  (user_b_id 에서)
--   4  (user_b_id 에서)

SELECT u.user_id, COUNT(A.user_id) AS num_friends
FROM users u
LEFT JOIN ALLID A ON u.user_id = A.user_id
GROUP BY u.user_id
ORDER BY num_friends DESC, user_id ASC;
```

```
핵심 포인트:
UNION ALL 을 쓴 이유 → 중복 제거 없이 양방향 관계를 전부 행으로 펼침
UNION 을 쓰면       → 같은 user_id 가 중복 제거돼서 친구 수가 줄어듦

양방향 관계 = UNION ALL 로 펼친 다음 COUNT
```

### 언제 이 패턴을 쓰는가

```
edges, follows, connections 같은 관계 테이블에서
양쪽 컬럼 모두를 기준으로 집계해야 할 때

예시:
- SNS 팔로우 수 집계
- 친구 관계 수 집계
- 거래 내역에서 입금/출금 양쪽 합산
- 네트워크 그래프에서 노드별 연결 수
```

---

## Code Core Points (4대 집합 연산자)

### 1. `UNION ALL` : 묻지도 따지지도 않고 다 붙이기 (가장 빠름)

두 쿼리의 결과를 중복이 있든 없든 **그대로 위아래로 이어 붙입니다.**

```sql
SELECT 사원명, 부서 FROM 서울지사
UNION ALL
SELECT 사원명, 부서 FROM 부산지사;
-- 결과: 만약 홍길동이 두 지사에 모두 등록되어 있다면, 홍길동이 2번(중복) 출력됨.
```

### `UNION` : 이어 붙이고 중복은 제거하기 (느림)

두 결과를 합친 다음, **내부적으로 `DISTINCT` 처럼 중복된 행을 하나로 합쳐서(제거해서)** 보여줍니다.

```sql
SELECT 사원명, 부서 FROM 서울지사
UNION
SELECT 사원명, 부서 FROM 부산지사;
-- 결과: 홍길동이 두 지사에 모두 있어도, 최종 결과에는 1명만 출력됨.
```

### `INTERSECT` : 양쪽에 다 있는 것만 (교집합)

위쪽 쿼리에도 있고, 아래쪽 쿼리에도 공통으로 존재하는 데이터만 뽑아낸다.

```sql
SELECT 사원명 FROM 상반기_우수사원
INTERSECT
SELECT 사원명 FROM 하반기_우수사원;
-- 결과: 상/하반기 모두 우수사원으로 뽑힌 찐 에이스 명단만 출력됨.
```

**중복 제거 ⭐️** 공통으로 존재하는 행이 여러 개 있어도 **하나의 행으로만 출력**한다.

```sql
-- 상반기에 '홍길동' 이 3번, 하반기에 2번 있어도
SELECT 사원명 FROM 상반기_우수사원
INTERSECT
SELECT 사원명 FROM 하반기_우수사원;
-- 결과: 홍길동 (1번만 출력 — 중복 제거됨)

-- 중복을 유지하고 싶다면 INTERSECT ALL 사용 (PostgreSQL 지원)
```

### `EXCEPT` / `MINUS` : 위에만 있는 것 (차집합)

위쪽 쿼리 결과에서 아래쪽 쿼리 결과를 빼고 남은 것만 보여줍니다. (💡 방언 주의: PostgreSQL, SQL Server는 `EXCEPT`를 쓰고, Oracle은 `MINUS`를 씁니다.)

```sql
SELECT 사원명 FROM 전체사원
EXCEPT   -- (Oracle이라면 MINUS)
SELECT 사원명 FROM 징계사원;
-- 결과: 징계받은 적 없는 깨끗한 사원 명단만 남음.
```

**EXCEPT / MINUS 도 중복을 제거한다 ⭐️**

차집합 연산 후 남은 결과에서도 **자동으로 중복 행을 제거**한다.

```sql
-- 전체사원에 '홍길동' 이 3번 들어있고
-- 징계사원에 없다면?

SELECT 사원명 FROM 전체사원
MINUS
SELECT 사원명 FROM 징계사원;

-- 결과: 홍길동 (1번만 나옴 — 중복 제거됨)
```

|연산자|중복 제거|비고|
|---|:-:|---|
|`UNION`|✅|합집합 + 중복 제거|
|`UNION ALL`|❌|합집합 + 중복 유지|
|`INTERSECT`|✅|교집합 + 중복 제거|
|`EXCEPT` / `MINUS`|✅|차집합 + 중복 제거|

> **`EXCEPT ALL` / `MINUS ALL`** 을 쓰면 중복을 유지한 채로 차집합을 구할 수 있다. PostgreSQL 은 `EXCEPT ALL` 지원, Oracle `MINUS ALL` 은 미지원.

---

## 집합 연산자 우선순위 (실행 순서)

> **SQL에서는 위에서 정의된(먼저 나온) 연산자가 먼저 수행된다.** 단, `INTERSECT`는 예외로 다른 연산자보다 **무조건 먼저** 실행된다.

|순위|연산자|비고|
|---|---|---|
|**1순위**|`INTERSECT`|항상 가장 먼저 실행|
|**2순위**|`UNION`, `UNION ALL`, `EXCEPT` / `MINUS`|위에서 아래 순서대로 실행|

**⚠️ 우선순위 때문에 결과가 달라지는 예시**

```sql
-- 🔴 의도: A에서 B를 빼고, 남은 결과에 C를 합치려 했음
-- ❌ 실제 동작: INTERSECT(B, C)를 먼저 계산하고, 그 결과를 A에서 뺌 → 의도와 다른 결과!
SELECT 사원명 FROM A
EXCEPT
SELECT 사원명 FROM B
INTERSECT         -- ← INTERSECT가 EXCEPT보다 먼저 실행됨!
SELECT 사원명 FROM C;
```

**✅ 괄호로 실행 순서를 명확히 지정하자**

```sql
(
    SELECT 사원명 FROM A
    EXCEPT
    SELECT 사원명 FROM B
)
INTERSECT
SELECT 사원명 FROM C;
-- 결과: A에서 B를 뺀 나머지와 C의 교집합 → 의도한 대로 작동!
```

---

## Detailed Analysis (집합 연산의 4대 절대 규칙)

### 컬럼의 "개수"가 똑같아야 한다!

- 위쪽이 3개면, 아래쪽 쿼리도 무조건 3개를 `SELECT` 해야 합니다.

### 컬럼의 "데이터 타입(순서)"이 서로 호환되어야 한다!

- 위쪽 첫 번째 컬럼이 숫자(`사원번호`)면, 아래쪽 첫 번째 컬럼도 숫자형이어야 합니다.

### `ORDER BY`는 맨 마지막에 딱 한 번만!

- 다 합쳐진 최종 결과물의 맨 밑에 딱 한 번만 써야 합니다.

### 최종 컬럼명은 무조건 "첫 번째(위쪽) 쿼리"를 따라간다!

- 아래쪽 쿼리에서 `AS`로 별칭을 아무리 예쁘게 지어봤자 100% 무시됩니다.

```sql
-- ⭕ 올바른 예시
SELECT 이름 AS 성명, 나이 FROM 학생
UNION ALL
SELECT 강사명 AS 이름, 나이 FROM 교사  -- 여기서 지은 '이름' 별칭은 무시됨
ORDER BY 나이 DESC;

-- ❌ 틀린 예시 (에러 발생)
SELECT 이름, 나이 FROM 학생 ORDER BY 나이  -- 🚨 여기서 정렬 불가
UNION ALL
SELECT 이름, 나이, 과목 FROM 교사;         -- 🚨 컬럼 개수 다름
```

---

## UNION (집합 연산) vs JOIN

|**구분**|**UNION (집합 연산)**|**JOIN (조인)**|
|---|---|---|
|**결합 방향**|수직 (위아래로 쌓기)|수평 (양옆으로 이어 붙이기)|
|**결과물 변화**|**행(Row)의 개수**가 늘어남|**열(Column)의 개수**가 늘어남|
|**전제 조건**|컬럼 개수와 타입이 같아야 함|공통 키(Key)가 필요함|
|**주요 용도**|분할된 테이블 병합, 양방향 관계 펼치기|서로 다른 정보 연결|

---

## 흔한 실수 (Common Beginner Misconceptions)

**"그냥 깔끔하게 중복 다 빼주는 `UNION`만 쓰면 안 되나요?"**

- `UNION`은 중복을 제거하기 위해 내부적으로 **정렬(Sort)** 작업을 수행합니다. 데이터가 많으면 성능에 큰 영향을 줍니다.
- 명백하게 중복이 없거나, 중복 제거가 필요 없다면 **무조건 `UNION ALL`을 써서 성능을 아껴야 합니다.**

**"양방향 관계 테이블에서 JOIN 했는데 숫자가 이상해요"**

- edges 같은 양방향 관계 테이블은 JOIN 이 아니라 **UNION ALL 로 펼친 다음 COUNT** 해야 합니다.
- 위의 🔥 실전 예제 참고.