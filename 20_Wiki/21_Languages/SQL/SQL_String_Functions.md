---
aliases:
  - SQL 문자열 함수
  - 문자열 조작
  - SUBSTR
  - LTRIM
  - REPLACE
  - SPLIT
  - STRING_AGG
  - GROUP_CONCAT
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Numeric_Functions]]"
  - "[[SQL_Date_Functions]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[SQL_Type_Casting]]"
---
# 문자열 자르고 붙이고 청소하기

>"사용자가 입력한 데이터는 생각보다 훨씬 더럽습니다. 
>띄어쓰기 오류, 대소문자 혼용 등을 깔끔하게 정제(Cleansing)하는 데이터 엔지니어의 필수 도구들입니다."


----
## **왜 필요한가?** 

현업의 Raw 데이터는 완벽하지 않다. 회원가입 이메일의 대소문자가 섞여 있거나, 쉼표(,)로 뭉쳐 있는 등 데이터 포맷이 파편화되어 있기 때문에, 이를 정제해야 정확한 데이터 분석과 JOIN이 가능하다.

---
## **실무에서 언제 쓰는가?**

- **데이터 정제 (Cleansing):** 공백 제거나 특수문자 제거 (`TRIM`, `REPLACE`).
- **데이터 추출:** 주민번호나 전화번호에서 특정 자리만 뽑아낼 때 (`SUBSTR`).
- **데이터 분할 및 병합:** 하나의 컬럼을 여러 행/열로 쪼개거나 (`SPLIT`), 반대로 여러 행을 한 줄로 예쁘게 말아올릴 때 (`STRING_AGG`).

---
## 대소문자 통일하기 (검색의 기본)

회원가입 시 입력받은 이메일이나 아이디를 비교할 때, 양쪽을 모두 대문자나 소문자로 통일해 놓고 비교(`{text}=`)해야 누락이 발생하지 않습니다.

- **`UPPER(문자열)`**: 모두 **대문자**로 바꿉니다. (예: `UPPER('sql')` ➡️ `'SQL'`)
- **`LOWER(문자열)`**: 모두 **소문자**로 바꿉니다. (예: `LOWER('SQL')` ➡️ `'sql'`)

---
## 공백 및 특정 문자 제거 (Trimming)

사용자가 실수로 입력한 스페이스바(공백)를 제거할 때 가장 많이 씁니다.

- **`LTRIM(문자열, [지울문자])`**: **왼쪽** 공백이나 지정한 문자를 지웁니다.
- **`RTRIM(문자열, [지울문자])`**: **오른쪽** 공백이나 지정한 문자를 지웁니다.
- **`TRIM([방향] [지울문자] FROM 문자열)`**: **양쪽(기본값)** 끝의 공백이나 특정 문자를 지웁니다. 방향에는 `LEADING`(왼쪽), `TRAILING`(오른쪽), `BOTH`(양쪽)를 쓸 수 있습니다.

**SQL Server의 배신**
- **Oracle & PostgreSQL:** `LTRIM`, `RTRIM`을 쓸 때 **'특정 문자'를 지정해서 지울 수 있습니다.**
- **SQL Server:** `LTRIM`, `RTRIM` 함수는 두 번째 인자를 받을 수 없으며, **오직 '공백(Space)'만 제거할 수 있습니다!** (특정 문자를 지우려면 아주 복잡하게 우회해야 합니다.)

**가다가 막히면 그 즉시 멈춘다! (연속성의 법칙)& "단어"가 아니라 "각각의 문자"다!**
- `LTRIM`과 `RTRIM`은 끝에서부터 문자를 지워 나가다가, **내가 지우라고 지시하지 않은 '다른 문자'를 만나는 순간 그 즉시 작업을 멈추고 종료**합니다.
- **여기서 가장 많이 하는 착각!** 두 번째 인자로 `'SQL'`을 주면 `"SQL"`이라는 단어가 통째로 붙어있어야 지워지는 게 아닙니다. **"S" 또는 "Q" 또는 "L" 중 하나라도 걸리면 무조건 지웁니다.** (순서 무관!)


```sql
-- 공백 제거 (모든 DB 공통 지원)
-- 결과: 'SQLD' (왼쪽 공백 3칸이 깔끔하게 날아감)
SELECT LTRIM('   SQLD') FROM DUAL;

-- 특정 문자('x') 제거하기 (DB별 차이점)

-- ✅ [Oracle / PostgreSQL] 정상 작동
-- 결과: 'SQLD' (왼쪽에 연속된 x를 모두 지움)
SELECT LTRIM('xxxSQLD', 'x') FROM DUAL; 

-- ❌ [SQL Server] 에러 발생! 
-- LTRIM 함수에는 인자가 1개(문자열)만 들어가야 한다는 에러가 뜹니다.
SELECT LTRIM('xxxSQLD', 'x'); 

-- ✅ [표준 TRIM 문법] 특정 문자 지우기의 정석 (Oracle / PostgreSQL)
-- 결과: 'SQL' (양쪽의 x를 모두 지움)
SELECT TRIM(BOTH 'x' FROM 'xxSQLxx') FROM DUAL;


-- 🎯 목표: 오른쪽 끝에 있는 'S' 또는 'Q' 또는 'L'을 지워라!

-- ✅ 결과: 'DB' 
-- 설명: 끝에서부터 'S', 'L', 'Q'가 뒤죽박죽 섞여 있지만 모두 타겟에 포함되므로 낱개로 하나씩 다 지워집니다. 
-- 그러다 'B'라는 장벽(모르는 문자)을 만나는 순간 즉시 멈춥니다!
SELECT RTRIM('DBSLQQSS', 'SQL') FROM DUAL; 

-- ✅ 결과: 'DBSQLxxY' 
-- 설명: 오른쪽 끝에서 'x' 2개를 지우며 신나게 전진하다가, 'Y'를 만나자마자 멈춥니다! 
-- 중간에 있는 'xx'는 껑충 뛰어넘지 못하고 그대로 살아남습니다.
SELECT RTRIM('DBSQLxxYxx', 'x') FROM DUAL;
```


---
## 문자열 자르기와 길이 재기 

주민등록번호에서 성별을 추출하거나, 전화번호 앞자리만 잘라낼 때 사용합니다.

### ① `SUBSTR` (문자열 자르기)

- **문법:** `SUBSTR(문자열, 시작위치, [자를길이])`
- **설명:** 지정한 위치부터 원하는 개수만큼 문자를 잘라옵니다. 길이를 생략하면 끝까지 다 가져옵니다.

**(DBMS별 차이점):**
- **Oracle:** `SUBSTR('SQLD', 2, 2)` ➡️ `'QL'` (2번째 글자부터 2개)
- **SQL Server:** `SUBSTRING('SQLD', 2, 2)` (이름이 다릅니다!)
- **PostgreSQL:** `SUBSTR()`과 `SUBSTRING()` **둘 다 완벽하게 지원**
- **음수 인덱스 (Oracle 전용):** `SUBSTR('SQLD', -2, 2)` ➡️ 뒤에서 2번째 글자부터 2개 가져오므로 `'LD'`가 됩니다.
- **PostgreSQL 실무 팁:** PostgreSQL은 `SUBSTR`에 음수 인덱스를 넣으면 오라클처럼 작동하지 않습니다! 대신 뒤에서부터 문자를 자르고 싶을 때는 직관적으로 **`RIGHT('SQLD', 2)`** 함수를 쓰는 것이 실무 국룰입니다. (앞에서 자를 땐 `LEFT()` 사용)

### ② LENGTH — 문자열 길이 재기

**문법:** `LENGTH(문자열)`
**설명:** 글자가 총 몇 개인지 **숫자**로 반환합니다.

> [!info] DBMS별 함수 비교
>
> | DB | 함수 | 예시 | 결과 |
> |:--|:--|:--|:--|
> | Oracle | `LENGTH()` | `LENGTH('SQLD')` | `4` |
> | PostgreSQL | `LENGTH()` | `LENGTH('SQLD')` | `4` |
> | PostgreSQL (ANSI 표준) | `CHAR_LENGTH()` | `CHAR_LENGTH('SQLD')` | `4` |
> | SQL Server | `LEN()` | `LEN('SQLD')` | `4` |

> [!warning] SQL Server는 `LEN` 을 씁니다!
> Oracle · PostgreSQL은 `LENGTH`, SQL Server만 **`LEN`** (끝에 GTH 없음)
> 시험에서 자주 나오는 함정입니다.

> [!tip] 줄바꿈 문자도 길이에 포함됩니다
> `'SQL\nD'` → LENGTH 결과는 `4`가 아니라 **`5`**
> 줄바꿈(`\n`), 탭(`\t`) 같은 **공백 문자도 1글자로 카운트**됩니다.
---

---
##  치환과 채워 넣기 (정제와 포매팅)
### ① REPLACE — 찾아서 바꾸기

**문법:** `REPLACE(문자열, 찾을문자, 바꿀문자)`
**설명:** 엑셀의 '찾기 및 바꾸기'와 완벽히 똑같습니다.

> **실무 예시**
> `REPLACE('010-1234-5678', '-', '')` → `'01012345678'` (하이픈 제거)


```sql
-- 1) 정상적인 치환 (모든 DB 공통)
-- 결과: '010*1234*5678' (하이픈을 별표로 바꿈)
SELECT REPLACE('010-1234-5678', '-', '*') FROM DUAL;

-- 2) 바꿀 문자를 생략하여 삭제 (Oracle 전용)
-- 결과: '01012345678' (하이픈 완전 제거)
SELECT REPLACE('010-1234-5678', '-') FROM DUAL;

-- 3) SQL Server / PostgreSQL: 반드시 빈 문자열('') 명시
-- 인자가 무조건 3개 있어야 함. 생략하면 문법 에러!
SELECT REPLACE('010-1234-5678', '-', '');
```

> [!warning] ⭐ SQLD 빈출 — DB별 3번째 인자 생략 가능 여부
>
> | DB | 3번째 인자 생략 | 결과 |
> |:--|:--|:--|
> | **Oracle** | ✅ 생략 가능 | 해당 문자를 완전히 **삭제** |
> | **PostgreSQL** | ❌ 생략 불가 | 문법 에러 → `''` 명시 필요 |
> | **SQL Server** | ❌ 생략 불가 | 문법 에러 → `''` 명시 필요 |

> [!tip] 💡 제거 스킬 암기법
>
> | 방식 | 코드 | 의미 |
> |:--|:--|:--|
> | **바꾸기** | `REPLACE(대상, 찾을것, 바꿀것)` | 찾아서 → 다른 걸로 교체 |
> | **지우기 (Oracle)** | `REPLACE(대상, 찾을것)` | 찾아서 → 흔적도 없이 삭제 |
> | **지우기 (그 외)** | `REPLACE(대상, 찾을것, '')` | 찾아서 → 빈 문자열로 교체 = 삭제 |
>
> 👉 **`CHR(10)` 응용:**
>
> | 코드 | 의미 |
> |:--|:--|
> | `CHR(10)` | ASCII 10번 = **줄바꿈 문자 (`\n`)** |
> | `REPLACE(C1, CHR(10))` | C1에서 줄바꿈을 찾아 → **삭제** (3번째 인자 생략, Oracle 전용) |
> | `REPLACE(C1, CHR(10), '')` | C1에서 줄바꿈을 찾아 → **삭제** (PostgreSQL / MSSQL) |

### ② `LPAD` / `RPAD` (빈자리 채우기)

- **문법:** `LPAD(문자열, 총길이, [채울문자])`
- **설명:** 정해진 길이만큼 문자를 만들되, 모자란 부분은 **왼쪽(L)** 또는 **오른쪽(R)** 에 특정 문자를 채워 넣습니다.
- **실무 예시:** 상품 코드나 사원 번호의 자릿수를 `0`으로 예쁘게 맞출 때 씁니다.
    - `LPAD('123', 5, '0')` ➡️ `'00123'` (총 5자리를 만드는데 왼쪽 빈칸을 0으로 채움)
    - `RPAD('AB', 4, '*')` ➡️ `'AB**'`

---
## 특정 기준으로 쪼개기 (SPLIT)

하나의 컬럼에 쉼표(`,`)나 이메일 골뱅이(`@`) 등으로 뭉쳐있는 데이터를 구분자(Delimiter)를 기준으로 여러 조각으로 쪼갤 때 사용합니다.
(표준 SQL이 아니어서 DBMS별로 문법 차이가 가장 심한 곳입니다.)

**DBMS별 차이점**

- **PostgreSQL (가장 직관적):** `SPLIT_PART(문자열, 구분자, 가져올조각번호)` 함수를 완벽 지원합니다.
- **Oracle:** 쪼개주는 전용 함수가 없어서 `REGEXP_SUBSTR` 같은 정규식 함수를 조합해 아주 복잡하게 잘라내야 합니다.
- **SQL Server:** `STRING_SPLIT(문자열, 구분자)` 함수를 쓰지만, 옆으로 나뉘는 게 아니라 테이블의 여러 행(Row)으로 세로로 분리되어 나옵니다.

```sql
-- 💡 1. [PostgreSQL] 가장 직관적인 SPLIT_PART 사용법
-- 회원테이블의 '이메일' 컬럼에서 '@'를 기준으로 쪼갠 뒤, 2번째 조각(도메인)만 추출
SELECT 
    사원명,
    이메일, 
    SPLIT_PART(이메일, '@', 2) AS 이메일_도메인 
FROM 사원테이블;
-- 결과: 'gmail.com', 'naver.com' 등


-- ⚠️ 2. [Oracle] 정규식(REGEXP_SUBSTR)을 활용한 쪼개기
-- '[^@]+' 뜻: '@'가 아닌 문자가 연속되는 덩어리 중에서, 1번째 문자부터 시작해 2번째 덩어리를 찾아라.
SELECT 
    사원명,
    이메일, 
    REGEXP_SUBSTR(이메일, '[^@]+', 1, 2) AS 이메일_도메인 
FROM 사원테이블;
-- 결과: 위 PostgreSQL과 동일하게 도메인만 추출됨


-- ⚠️ 3. [SQL Server] 세로(Row)로 쪼개지는 STRING_SPLIT 사용법
-- '취미목록' 컬럼에 '독서,등산,게임' 처럼 쉼표로 뭉쳐있는 데이터를 행으로 분리한다.
-- 주의: 옆으로 컬럼이 늘어나는 게 아니라, 데이터 행 자체가 밑으로 여러 개 늘어난다.
SELECT 
    사원명, 
    value AS 개별_취미 
FROM 사원테이블
    CROSS APPLY STRING_SPLIT(취미목록, ',');
/* 결과: 
사원명  | 개별_취미
----------------
홍길동  | 독서
홍길동  | 등산
홍길동  | 게임
*/
```

---
## String Aggregation (여러 행을 하나의 문자열로 합치기) `STRING_AGG`,`GROUP_CONCAT`

`SPLIT`의 정확히 반대되는 개념이다. 
`GROUP BY`로 묶인 여러 행(Row)의 데이터를 쉼표(,) 등의 구분자를 넣어 하나의 문자열(Column)로 예쁘게 말아올린다. 부서별 사원 목록이나, 유저별 구매 상품 목록을 한 줄로 뽑아낼 때 필수적이다.

### 문법 및 내부 구조

- **PostgreSQL:** 함수 괄호 **안에** 전부 넣는다. 
	- `{sql}STRING_AGG([DISTINCT] 합칠컬럼, '구분자' [ORDER BY 정렬컬럼 ASC|DESC])`
- **MySQL:** 함수 괄호 **안에** 전부 넣되, 구분자는 `SEPARATOR` 키워드를 쓴다. 
	- `GROUP_CONCAT([DISTINCT] 합칠컬럼 [ORDER BY 정렬컬럼] [SEPARATOR '구분자'])`
-  **Oracle:** 함수 괄호 **밖에서** `WITHIN GROUP`으로 정렬한다. (DISTINCT는 정규식 등 별도 처리 필요) 
	- `LISTAGG(합칠컬럼, '구분자') WITHIN GROUP (ORDER BY 정렬컬럼)`
- **SQL Server:** Oracle처럼 함수 괄호 **밖에서** `WITHIN GROUP`으로 정렬한다.
	- `STRING_AGG(합칠컬럼, '구분자') WITHIN GROUP (ORDER BY 정렬컬럼)`

```sql
-- 💡 1. [PostgreSQL] STRING_AGG 사용법
-- 괄호 안에서 중복(DISTINCT)도 제거하고, 이름순(ORDER BY)으로 정렬까지 한 번에 처리한다.
SELECT 
    부서명, 
    STRING_AGG(DISTINCT 사원명, ', ' ORDER BY 사원명 ASC) AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;
-- 결과: 영업부 | '김철수, 이영희, 홍길동'


-- 💡 2. [MySQL] GROUP_CONCAT 사용법 
-- PostgreSQL과 비슷하지만 SEPARATOR 키워드를 명시해야 한다.
SELECT 
    부서명, 
    GROUP_CONCAT(DISTINCT 사원명 ORDER BY 사원명 ASC SEPARATOR ', ') AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;


-- ⚠️ 3. [Oracle] LISTAGG 사용법
-- WITHIN GROUP을 사용하여 그룹 내 정렬 방식을 함수 바깥에서 지정한다.
SELECT 
    부서명, 
    LISTAGG(사원명, ', ') WITHIN GROUP (ORDER BY 사원명 ASC) AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;


-- ⚠️ 4. [SQL Server] STRING_AGG 사용법 (2017 버전 이상)
-- Oracle과 동일하게 WITHIN GROUP을 사용해 정렬한다.
SELECT 
    부서명, 
    STRING_AGG(사원명, ', ') WITHIN GROUP (ORDER BY 사원명 ASC) AS 부서원_목록
FROM 사원테이블
GROUP BY 부서명;
```

>`GROUP BY`로 묶인 그룹 내의 지정된 컬럼 값들을 차례대로 순회하며, 지정한 구분자를 사이에 끼워 넣어 하나의 긴 문자열로 반환한다.
>`ORDER BY`를 쓰면 A-Z, 가-나-다 순 등으로 예쁘게 정렬된 문자열을 얻을 수 있다.

---
## 컴퓨터의 언어로 변환 (ASCII)

- **`CHAR` / `CHR`**: 숫자(ASCII 코드값)를 던져주면, 그 숫자에 해당하는 **문자**를 반환합니다.

**DBMS별 차이점**
- **Oracle & PostgreSQL:** `CHR(65)` ➡️ `'A'` (PostgreSQL도 오라클처럼 `CHR`을 씁니다!)
- **SQL Server:** `CHAR(65)` ➡️ `'A'`
- **공통점:** 반대로 문자를 ASCII 코드로 바꿀 때는 **세 DB 모두 공통으로 `ASCII('A')` ➡️ `65`** 를 사용합니다.

