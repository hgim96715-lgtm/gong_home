---
aliases:
  - SQL 정렬
  - ORDER BY
  - 오름차순
  - 내림차순
  - NULL 정렬
  - 커스텀 정렬
tags:
  - SQL
  - Sorting
related:
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Filtering_WHERE]]"
---
## 개념 한 줄 요약

**ORDER BY 절**은 뒤죽박죽 섞여 있는 데이터를 **특정 기준에 맞춰 줄 세우는(Sorting)** 문법이야. 
(엑셀의 '정렬' 버튼과 똑같아!)

---
## 왜 필요한가 (Why)

 **문제점:**
 - SQL 데이터베이스는 데이터를 저장할 때 순서를 보장하지 않아.
 - (`SELECT` 할 때마다 순서가 바뀔 수도 있음!)
 
 **해결책:** 
 - "가입일 순서대로 보여줘", "매출 높은 순으로 10명만 뽑아줘"처럼 **명확한 순서**가 필요할 때 필수야.
 - 특히 `LIMIT`와 함께 써서 **랭킹(Ranking)** 을 구할 때 가장 중요해.

---
## Basic Syntax (기본기)

### A. 오름차순 vs 내림차순

* **`ASC` (Ascending):** 작은 것 ➡ 큰 것 (1, 2, 3... / A, B, C... / 옛날 ➡ 최신)
    * **기본값(Default)** 이라 생략 가능하지만, 명시해주는 게 가독성에 좋아.

* **`DESC` (Descending):** 큰 것 ➡ 작은 것 (3, 2, 1... / Z, Y, X... / 최신 ➡ 옛날)

```sql
-- 매출 높은 순(내림차순)으로 정렬
SELECT * FROM Sales
ORDER BY amount DESC;
```

### B. 다중 정렬 (Multi-level Sorting)

기준이 여러 개일 때, **앞에 쓴 컬럼이 우선순위**를 가져.

```sql
-- 1차: 부서 이름 순(ASC)으로 줄 세우고,
-- 2차: 같은 부서라면 월급 높은 순(DESC)으로 정렬
ORDER BY department ASC, salary DESC;
```

---
## Advanced Techniques (심화)

### A. NULL 값의 위치 제어 (NULLS FIRST / LAST)

**데이터 엔지니어 면접 단골 질문!** "정렬할 때 NULL은 어디로 갈까요?"

**명시적 제어 (표준 문법):**
- 오름차순(ASC)이든 내림차순(DESC)이든 상관없이 **NULL의 위치를 강제로 고정**하는 명령어
- **`NULLS FIRST`**: NULL 값을 무조건 **맨 위(처음)** 로 가져와.
- **`NULLS LAST`**: NULL 값을 무조건 **맨 아래(마지막)** 로 보내버려.

**DB마다 기본값이 다름 (설정 안 했을 때):**
- **MySQL:** NULL을 **가장 작은 값** 취급 (ASC 하면 맨 위, DESC 하면 맨 아래).
- **PostgreSQL/Oracle:** NULL을 **가장 큰 값** 취급 (ASC 하면 맨 아래, DESC 하면 맨 위).

**해결책:**
- DB마다 기본 동작이 다르니 헷갈리지 않게 **직접 지정**해주는 게 제일 안전해.

```sql
-- 오름차순(1, 2, 3...)으로 보되,
-- 값이 없는(NULL) 데이터는 맨 뒤(LAST)로 보내줘!
ORDER BY commission_pct ASC NULLS LAST;
```

(참고: MySQL은 `NULLS LAST` 문법을 지원 안 해. 대신 `ORDER BY column IS NULL ASC` 같은 방식을 써야 함.)

### B. 별칭(Alias)과 숫자 사용

- **별칭 사용 가능:** `SELECT` 절에서 만든 별칭(`AS`)을 `ORDER BY`에서 바로 쓸 수 있어.
- (실행 순서상 가능!)

```sql
SELECT price * quantity AS total_amount
FROM Orders
ORDER BY total_amount DESC; -- OK!
```

- **숫자(위치) 사용:** "몇 번째 컬럼"인지 숫자로 지정 가능.

```sql
SELECT name, age, city
FROM Users
ORDER BY 2 DESC; -- 2번째 컬럼인 'age'로 정렬해라.
```

**⚠️ 주의:** 숫자로 정렬하는 건 **실무에서 비추천!** 
나중에 `SELECT` 컬럼 순서를 바꾸면 정렬 기준이 엉뚱하게 바뀌어 버려서 유지보수에 최악이야. 
(그냥 컬럼명을 쓰자!)

### C. 내 맘대로 정렬 (Custom Sorting)

"사장, 부장, 대리, 사원" 순으로 정렬하고 싶은데, 가나다순으로 하면 "대리, 부장, 사원, 사장"이 되어버리잖아? 
이럴 땐 **`CASE WHEN`** 으로 순서를 직접 매겨야 해.

```sql
ORDER BY 
  CASE position
    WHEN 'CEO' THEN 1
    WHEN 'Manager' THEN 2
    WHEN 'Staff' THEN 3
    ELSE 4 
  END ASC;
```

---
## 초보자가 자주 착각하는 포인트

1. **"정렬 안 해도 순서대로 나오던데?"**
	- 우연일 뿐이야! 데이터가 삭제되고 추가되다 보면 물리적인 저장 순서가 꼬여서 언제든 뒤섞일 수 있어.
	- 순서가 중요하다면 **무조건 `ORDER BY`를 명시**해야 해.

2.  성능 문제 (Performance):
	- 정렬은 컴퓨터한테 굉장히 힘든 작업(비용이 큼)이야.
	- 데이터가 수백만 건인데 인덱스(Index) 없이 `ORDER BY`를 하면 쿼리가 엄청 느려질 수 있어. 
	- 나중에[[Query_Optimization]]에서 다룰 예정!