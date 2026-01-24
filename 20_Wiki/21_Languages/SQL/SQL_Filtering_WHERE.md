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
tags:
  - SQL
  - Filtering
related:
  - "[[SQL_SELECT_FROM]]"
  - "[[HAVING_vs_WHERE]]"
---
## 개념 한 줄 요약

**WHERE 절**은 엑셀의 '필터' 기능과 같아.
테이블에 있는 수많은 데이터 중에서 **내가 원하는 조건에 맞는 행(Row)만 쏙 뽑아내는 문법**이야.

---
## 왜 필요한가 (Why)

 **문제점:** 
* `SELECT * FROM Orders`를 하면 1억 건의 주문 데이터가 다 나와서 컴퓨터가 멈출 수도 있어.

 **해결책:** 
 - "서울 사는 사람만", "어제 주문한 것만" 처럼 조건을 걸어서 필요한 데이터만 가져와야 해. 
 - (비용 절감 & 속도 향상)

---
## Core Operators (필수 암기)

### A. 비교 연산자 (Comparison)

가장 기본적인 크기 비교야.

* `{text}=`: 같다 (Equal)
* `<>` 또는 `!=`: 다르다 (Not Equal)
* `>`, `<`, `>=`, `<=`: 크다, 작다 등

### B. 논리 연산자 (Logical)

조건이 여러 개일 때 연결해주는 접착제.

* `AND`: 두 조건이 **모두** 참이어야 함 (교집합)
* `OR`: 둘 중 **하나라도** 참이면 됨 (합집합)
* `NOT`: 조건의 반대 (여집합)

>**🚨 드 모르간의 법칙 (De Morgan's Laws)**
>`NOT`이 괄호 안으로 들어가면 **AND는 OR로, OR는 AND로 바뀐다.** (이거 모르면 나중에 데이터가 막 섞여!)
> **`NOT (A OR B)`** ➡ **`NOT A AND NOT B`**
>  예: "주말(토 또는 일)이 아닌 날" = "토요일 아님 **그리고** 일요일 아님"
> **`NOT (A AND B)`** ➡ **`NOT A OR NOT B`**

> **💡 실전 응용: 
> 불량 데이터 한 방에 청소하기** 
> 데이터 분석을 할 때, 여러 컬럼 중에 **하나라도 NULL(빈 값)이 있으면** 분석에서 제외해야 할 때가 많아. 
>  이때 드 모르간의 법칙을 쓰면 코드가 아주 직관적으로 변해.

>**상황:** 종(species)이나 몸무게(body_mass_g) 정보가 없는 펭귄은 분석에서 빼고 싶다!

```sql
-- 1. 드모르간 법칙 : "나쁜 데이터(NULL)들을 묶어서 버리자(NOT)."
WHERE NOT (species IS NULL OR body_mass_g IS NULL)

-- 2. (동일한 결과): "온전한 데이터(NOT NULL)만 남기자(AND)."
WHERE species IS NOT NULL AND body_mass_g IS NOT NULL

-- 결과적으로 둘은 같은 말이야.
```


### C. 범위 연산자 (Range - BETWEEN)

"A부터 B까지"를 표현할 때 사용해.

- **문법:** `WHERE age BETWEEN 10 AND 20`
- **특징:** 양 끝값(10과 20)을 **모두 포함(Inclusive)**해. (`>= 10 AND <= 20`과 동일)

> **⚠️ 날짜(Date) 사용 시 주의점!**
> `BETWEEN '2024-01-01' AND '2024-01-02'`라고 쓰면 `2024-01-02 00:00:00`까지만 가져와.
> 만약 `2024-01-02 14:30:00` 데이터가 있다면 누락돼.
> 실무에서는 안전하게 `date >= '2024-01-01' AND date < '2024-01-03'` 방식을 더 선호하기도 해.

###  D. 집합 연산자 (IN)

"이거거나, 이거거나, 이거거나" 할 때 `OR`를 여러 번 쓰면 코드가 지저분해지잖아?

* **`IN (A, B, C)`**: A 또는 B 또는 C인 경우. (`OR`의 단축 버전)
* `NOT IN (...)`: 저것들을 제외한 나머지. 
	* **주의: 목록 안에 NULL이 하나라도 있으면 결과가 아무것도 안 나옴!**

```sql
-- OR 사용 (비추천)
WHERE category = 'Furniture' OR category = 'Office'

-- IN 사용 (추천)
WHERE category IN ('Furniture', 'Office')
```

### E. 패턴 매칭 (LIKE / ILIKE) & ESCAPE

정확한 단어가 아니라 "김씨로 시작하는 사람", "gmail을 쓰는 사람"을 찾을 때 써.

**`LIKE`**: 대소문자를 구분함. (`'Apple'` != `'apple'`)

- `%`: 와일드카드 (모든 문자). `김%` (김으로 시작하는 모든 것)
- `_`: 한 글자. `김_수` (김O수 인 사람)

**`ILIKE`**: 대소문자 구분 안 함. (PostgreSQL, Redshift 등에서 사용. MySQL은 기본적으로 구분 안 함)

- `WHERE email ILIKE '%@gmail.com'` -> 대문자 GMAIL도 다 찾아줌. **(실무 꿀팁!)**

> **🔦 특수문자 검색하기 (ESCAPE)**
> 만약 데이터 내용 중에 진짜 `%`나 `_`가 들어있어서 그걸 찾고 싶다면?
> 그냥 `%`를 쓰면 와일드카드로 인식되니까, 앞에 **탈출 문자(Escape Character)** 를 붙여줘야 해.

```sql
-- '50%'라는 글자가 포함된 데이터를 찾고 싶을 때
-- 역슬래시(\) 뒤에 오는 %는 문자로 취급해라! 라는 뜻
WHERE feedback LIKE '%50\%%' ESCAPE '\';
```

### E. 결측치 확인 (IS NULL) 🚨 중요!

**SQL에서 가장 많이 틀리는 부분 1위.**

- `NULL`은 '0'이나 '공백'이 아니라 **"값이 없음(Unknown)"** 상태야.
- 그래서 `{text}=` 연산자로 비교가 불가능해. (`NULL = NULL`은 False임)
- **반드시 `IS NULL` 또는 `IS NOT NULL`을 써야 해.**

```sql
-- 틀린 예시 (아무것도 안 나옴)
WHERE phone_number = NULL 

-- 맞는 예시
WHERE phone_number IS NULL
```

---
##  Critical Rules (절대 안 되는 것들) 

**"WHERE 절에서 에러가 나요!"** 하는 경우의 90%는 이 두 가지 규칙을 어겨서 그래. 
SQL이 실행되는 순서 때문에 발생하는 문제니 꼭 외워둬.

### A. 별칭(Alias) 사용 불가

`SELECT` 절에서 만든 별칭(`AS`)은 `WHERE` 절에서 쓸 수 없어.
* **이유:** SQL은 위에서 아래로 실행되는 게 아니야. 
* **`FROM` ➡ `WHERE` ➡ `SELECT`** 순서로 실행돼. 즉, `WHERE`가 실행될 시점에는 아직 별칭이 만들어지지도 않았어.

```sql
-- ❌ 틀린 예시 (에러 발생: Unknown column 'total')
SELECT price * quantity AS total
FROM Orders
WHERE total > 1000; 

-- ✅ 맞는 예시 (원본 컬럼으로 계산)
SELECT price * quantity AS total
FROM Orders
WHERE price * quantity > 1000;
```

(참고: `ORDER BY`는 `SELECT`보다 나중에 실행돼서 별칭 사용이 가능해!)

### B. 집계 함수(Aggregate Function) 사용 불가

`SUM`, `AVG`, `COUNT`, `MAX` 같은 함수는 `WHERE` 절에 못 써.

 **이유:** 
- `WHERE`는 **행(Row) 하나하나**를 검사하는 녀석이야. 
- 그런데 `AVG(salary)`는 여러 행을 뭉쳐야 계산이 되잖아? 레벨이 안 맞아서 에러가 나.

 **해결책:** 
 - 집계 함수로 조건을 걸고 싶으면 `WHERE`가 아니라 **`HAVING`** 을 써야 해.

```sql
-- ❌ 틀린 예시 (Invalid use of group function)
SELECT department_id, AVG(salary)
FROM Employees
WHERE AVG(salary) >= 5000;

-- ✅ 맞는 예시 (HAVING 사용)
SELECT department_id, AVG(salary)
FROM Employees
GROUP BY department_id
HAVING AVG(salary) >= 5000;
```

> "자세한 차이점은 **[[HAVING_vs_WHERE]]** 문서 참고!"

---
## 초보자가 자주 착각하는 포인트

1. **`NULL` 찾을 때 `{text}=` 쓴다:** 
	- 위에서 말했듯 절대 안 됨. 
	- 무조건 `IS` 문법 사용!

2.  **`AND`와 `OR`의 우선순위:**
	- 둘을 섞어 쓸 때는 헷갈리니까 **무조건 괄호 `()`를 쳐주는 습관**을 들이자.
	- 컴퓨터는 `AND`를 먼저 계산해버리거든.

3.  **문자열에 따옴표 안 붙임:** 
	- 숫자는 그냥 써도 되지만, 문자는 반드시 `'Text'` 처럼 작은따옴표를 붙여야 해.

4. **`BETWEEN`은 끝값을 뺀다?**:
	- 아니, 무조건 포함해. (이상 ~ 이하)

5. **`NOT IN`안에 NULL:**
	- `WHERE id NOT IN (1, 2, NULL)` 하면 결과가 0건 나와. 
	- (SQL 입장에서 NULL은 알 수 없는 값이라 제외 여부를 판단 못 해서 다 버림).

