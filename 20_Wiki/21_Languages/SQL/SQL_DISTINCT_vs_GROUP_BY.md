---
aliases:
  - DISTINCT
  - 중복제거
  - Unique
  - COUNT(DISTINCT 컬럼)
tags:
  - SQL
related:
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[00_SQL_HomePage]]"
---
# SQL DISTINCT vs GROUP BY (중복 제거와 그룹화)

## Summary (개념 한 줄 요약)

단순히 '종류(목록)'만 알고 싶으면 `SELECT DISTINCT`를, 데이터를 묶어서 통계를 내려면 `GROUP BY`를, 그룹 내에서 '서로 다른 고유한 종류의 개수'를 세려면 `COUNT(DISTINCT)`를 사용한다.

---
## Why it is needed (왜 필요한가?)

집계(숫자 계산)가 필요 없고 단순히 "어떤 조합이 있는지(Unique Combination)"만 궁금할 때 `GROUP BY`를 쓰면 쿼리 목적이 불분명해진다. 
단순히 목록만 추출할 때는 `DISTINCT`가 훨씬 직관적이고 빠르다. 반대로 그룹화된 통계가 필요할 때는 `GROUP BY`를 써야 한다.

---
## When to use in practice (실무 적용 시점)

* **목록(Catalog)이나 메뉴판 만들기:** "우리 쇼핑몰에 등록된 대분류/소분류 카테고리 목록만 중복 없이 뽑아주세요." (`SELECT DISTINCT`)
* **DAU(일일 활성 사용자 수) 계산:** 오늘 하루 쇼핑몰에 접속한 총 횟수가 아니라, '순수 방문자 수'를 구할 때. 철수가 10번 접속해도 1명으로 셈. (`COUNT(DISTINCT user_id)`)

---

## Core Mechanism (핵심 동작 원리)

### 3가지 핵심 문법 비교표

| 구분 | SELECT DISTINCT | GROUP BY | COUNT(DISTINCT) |
| :--- | :--- | :--- | :--- |
| **목적** | 목록 / 메뉴판 만들기 | 데이터 묶기 (통계용) | 고유값 개수 세기 |
| **사용 위치** | `SELECT` 바로 뒤 | `FROM` / `WHERE` 뒤 | `SELECT`나 `HAVING` 절 내부 |
| **작동 원리** | 조회된 행 전체의 중복을 가림 | 지정된 기준으로 데이터를 파티셔닝 | 그룹 내에서 중복을 뺀 개수만 셈 |

###  DISTINCT (A||B) 메커니즘 (문자열 결합)

`||`는 오라클과 PostgreSQL 등에서 사용하는 '문자열 결합' 연산자다. 두 컬럼을 하나의 문자열로 합친 뒤 중복을 제거하는 테크닉이다.
* 작동 방식: A 컬럼('홍')과 B 컬럼('길동')을 합쳐서 하나의 임시 컬럼('홍길동')으로 만든 뒤, 그 덩어리의 중복을 제거하거나 카운트한다.
* 예: `COUNT(DISTINCT colA || colB)`

---
## Core Code Explanation (핵심 코드 설명)

### 1. 단순 목록 추출 (SELECT DISTINCT)

집계 함수 없이 펭귄의 종(species)과 서식지(island)의 조합 목록만 뽑아낸다.

```sql
-- 지난달 주문 내역에서 배송이 갔던 '시/도'와 '구/군' 목록만 중복 없이 확인
SELECT DISTINCT city, district
FROM orders
ORDER BY city, district;
```

### 2. 고유 개수 조건 필터링 (HAVING + COUNT DISTINCT)

"DISTINCT는 단순히 목록을 뽑을 때도 쓰지만, 집계 함수(`COUNT`) 안에 쏙 들어갈 때 진짜 파괴력이 살아난다."

```sql
-- 서로 다른 종류의 펭귄 종(species)을 2개 이상 보유한 섬(island)만 찾아라!
SELECT island, COUNT(DISTINCT species) AS species_cnt
FROM penguins
GROUP BY island
HAVING COUNT(DISTINCT species) >= 2;
```

>섬(island)별로 데이터를 묶은 뒤, 그 섬에 사는 펭귄 데이터 중 이름이 같은 종은 1번만 카운트하여 그 고유 개수가 2 이상인 곳만 필터링한다.

---
## Memory / Exam Points (복습 메모 & 주의사항)

- 💡 **DISTINCT 문법 O/X 체크리스트 (위치를 정확히 기억할 것!)**
	- ✅ `SELECT DISTINCT col1, col2` **(O)** : 올바른 문법. 전체 조회 결과에서 행(Row) 단위로 중복을 제거한다.
	- ✅ `COUNT(DISTINCT col)` **(O)** : 올바른 문법. 괄호 안의 특정 컬럼만 중복을 제거한 뒤 개수를 센다.
	- ❌ `SELECT col1, DISTINCT(col2)` **(X)** : 절대 불가! DISTINCT는 엑셀 함수처럼 특정 일반 컬럼 하나에만 씌울 수 없다. SELECT 절에서는 무조건 맨 앞에 한 번만 적어야 한다.

- **핵심 주의사항: 중복 판단의 기준은 "Row 전체"다.**
	- `SELECT DISTINCT`를 쓸 때, 뒤에 적힌 컬럼들을 "하나의 세트"로 묶어서 본다.
	- (철수, 서울) vs (철수, 부산) 👉 이름은 같지만 도시가 다르므로 "다른 데이터" 취급 (둘 다 출력)
	- (철수, 서울) vs (철수, 서울) 👉 완전히 동일하므로 하나만 출력
	- **결론:** 만약 A 컬럼의 중복만 완벽하게 없애고 싶다면, 뒤에 B 컬럼을 같이 SELECT 하면 안 된다! (`SELECT DISTINCT A`만 써야 함)
