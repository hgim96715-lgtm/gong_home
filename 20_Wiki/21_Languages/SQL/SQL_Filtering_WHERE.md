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
---
## 개념 한 줄 요약 

**"엑셀의 '필터' 기능과 완전히 같습니다."** 
수많은 데이터 중에서 **내가 원하는 조건에 맞는 행(Row)만 쏙 뽑아내는 문법**입니다.

---
## 왜 필요한가요? (Why)

|**구분**|**설명**|
|---|---|
|**문제점 😱**|`SELECT * FROM Orders`를 하면 1억 건의 데이터가 쏟아져 나와 **서버가 다운**될 수 있습니다.|
|**해결책 ✅**|"서울 사는 사람", "어제 주문 건"처럼 **조건을 걸어 필요한 데이터만** 가져옵니다. (속도 향상 & 비용 절감)|

---
## 핵심 연산자 (Core Operators)

### A. 비교 연산자 (Comparison)

가장 기본적인 크기 비교입니다.

|**연산자**|**의미**|**예시**|
|---|---|---|
|**`=`**|같다|`age = 20`|
|**`<>`**, **`!=`**|다르다|`city != 'Seoul'`|
|**`>`, `<`**|크다, 작다|`price > 1000`|
|**`>=`, `<=`**|크거나 같다, 작거나 같다|`score <= 50`|

### B. 논리 연산자 (Logical)

조건이 여러 개일 때 연결해주는 접착제입니다.

- **`AND`**: 두 조건이 **모두** 참이어야 함 (교집합)
- **`OR`**: 둘 중 **하나라도** 참이면 됨 (합집합)
- **`NOT`**: 조건의 반대 (여집합)

**[필수 암기] 드 모르간의 법칙 (De Morgan's Laws)** `NOT`이 괄호 안으로 들어가면 **AND ↔ OR**로 바뀝니다.

- **`{sql}NOT (A OR B)`** ➡ **`{sql}NOT A AND NOT B`**
    - (예: "주말이 아닌 날" = "토요일 아님 **그리고** 일요일 아님")
- **`{sql}NOT (A AND B)`** ➡ **`{sql}NOT A OR NOT B`**

**💡 실무 꿀팁: 불량 데이터 청소** "종(Species)이나 몸무게(Mass) 중 하나라도 비어있으면(NULL) 버려라!"

```sql
-- 방법 1 (드 모르간 적용): 나쁜 놈들을 묶어서(OR) 부정(NOT)
WHERE NOT (species IS NULL OR body_mass_g IS NULL)

-- 방법 2 (동일 결과): 온전한 놈들만(NOT NULL) 남기자(AND)
WHERE species IS NOT NULL AND body_mass_g IS NOT NULL
```

### C. 범위 연산자 (BETWEEN)

"A부터 B까지"를 표현합니다.

- **문법:** `{sql}WHERE age BETWEEN 10 AND 20`
- **특징:** **양 끝값(10, 20)을 모두 포함(Inclusive)** 합니다. (`>= 10 AND <= 20`)

> **⚠️ 날짜(Date) 사용 시 주의!** 
> `BETWEEN '1일' AND '2일'`로 조회하면 **2일 00시 00분 00초**까지만 가져옵니다.
>  (2일 오후 데이터 누락됨). 실무에서는 부등호(`>=`, `<`)를 더 선호합니다.

### D. 집합 연산자 (IN)

`OR`를 여러 번 쓰는 노가다를 줄여줍니다.

- **`{sql}IN (A, B, C)`**: A 또는 B 또는 C인 경우.
- **`{sql}NOT IN (...)`**: 리스트에 없는 경우.
    - **💀 주의:** 목록 안에 `NULL`이 하나라도 있으면 **결과가 0건** 나옵니다. (SQL이 판단 포기)


### E. 패턴 매칭 (LIKE / ILIKE)

정확한 일치가 아니라 "포함된 단어"를 찾을 때 사용합니다.

- **`{sql}LIKE`**: 대소문자 구분 함. (`Apple` ≠ `apple`)
- **`{sql}ILIKE`** (PostgreSQL): 대소문자 구분 안 함. (`Apple` == `apple`)

- **와일드카드:**
- `%`: 모든 문자 (0글자 이상) -> `김%` (김, 김수, 김철수...)
- `_`: 딱 한 글자 -> `김_수` (김철수 O, 김영희수 X)

>**특수문자 검색 (ESCAPE)** 
>진짜 `%` 글자를 찾고 싶다면? 앞에 역슬래시(`\`)를 붙여주세요. 
>`{sql}WHERE review LIKE '%50\%%' ESCAPE '\'` (50%라는 글자를 찾아라)


### 결측치 확인 (IS NULL) 

**가장 많이 틀리는 부분 1위입니다.**

- `NULL`은 '0'도 '공백'도 아닌 **"알 수 없음"** 상태입니다.
- 따라서 `{text}=` 연산자로 비교가 불가능합니다.
- **무조건 `{sql}IS NULL` / `{sql}IS NOT NULL`을 써야 합니다.**

---
## 고급 패턴 매칭 (PostgreSQL 특화)

### Q. `IN` 안에 와일드카드(`%`)를 쓸 수 있나요?

**A. 아니요, 불가능합니다.**
`IN`은 오직 **값이 완전히 똑같은지(Exact Match)** 만 확인합니다.

```sql
-- ❌ 틀린 시도 (아무것도 안 나옴)
WHERE email IN ('%@gmail.com', '%@naver.com') 
-- 해석: 이메일 주소가 진짜로 "%@gmail.com"이라는 글자인 사람을 찾아라.
```

### Solution 1. 여러 패턴 한 번에 찾기 (`ILIKE ANY`)

`OR`를 여러 번 쓰는 대신, 배열(`ARRAY`)을 사용하여 `IN`처럼 깔끔하게 씁니다.

```sql
-- ✅ OR 반복 사용 (지저분함)
WHERE email ILIKE '%gmail%' OR email ILIKE '%naver%'

-- ✅ ILIKE ANY 사용 (깔끔!) ✨
-- "배열 안의 패턴 중 하나(ANY)라도 맞으면 가져와라"
WHERE email ILIKE ANY (ARRAY['%gmail%', '%naver%'])
```

### Solution 2. 정규표현식 사용 (Regular Expressions)

더 복잡하고 정교한 패턴을 찾을 때 사용합니다. (PostgreSQL 전용)

- **`~`**: 대소문자 구분 **함** (Case-sensitive)
- **`~*`**: 대소문자 구분 **안 함** (Case-insensitive) - **가장 많이 씀!**
- **`!~`**: 매칭되지 않는 것 (Not match)

```sql
-- 'fire' 또는 'water'가 포함된 단어 찾기 (대소문자 무시)
WHERE description ~* 'fire|water'

-- 이름이 'A'나 'B'로 시작하는 사람 찾기
WHERE name ~* '^(A|B)'
```

---
## 절대 안 되는 것들 (Critical Rules) 

SQL 실행 순서(`FROM` → `WHERE` → `SELECT`) 때문에 발생하는 에러입니다.

### A. 별칭(Alias) 사용 불가

`WHERE`는 별명을 짓기(`SELECT`) 전에 실행되므로, 별명을 모릅니다.

```sql
-- ❌ 에러: 'total'이 뭐야?
SELECT price * quantity AS total FROM Orders WHERE total > 1000; 

-- ✅ 정답: 원본 컬럼으로 계산
SELECT price * quantity AS total FROM Orders WHERE price * quantity > 1000;
```

### B. 집계 함수 사용 불가

`WHERE`는 개별 행을 검사하므로, 그룹 합계(`SUM`, `AVG`)를 알 수 없습니다.

```sql
-- ❌ 에러: WHERE 절엔 집계함수 불가
SELECT dept, AVG(salary) FROM Emp WHERE AVG(salary) > 5000;

-- ✅ 정답: HAVING 사용
SELECT dept, AVG(salary) FROM Emp GROUP BY dept HAVING AVG(salary) > 5000;
```

---
## 초보자가 자주 하는 실수 Check 

1. **`NULL` 찾을 때 `{sql}=` 씀** 👉 무조건 `IS NULL` 쓰세요.
2. **`AND`와 `OR` 혼용 시 괄호 안 침** 👉 `AND`가 먼저 계산되니 괄호 `()` 필수!
3. **문자열에 따옴표 안 붙임** 👉 숫자는 그냥, 문자는 `'Text'`
4. **`BETWEEN` 끝값 제외 착각** 👉 양 끝값 모두 포함입니다.
5. **`NOT IN` 안에 `NULL` 포함** 👉 결과가 통째로 증발합니다. (NULL 제외 필수)