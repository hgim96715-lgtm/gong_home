---
aliases:
  - SQL NULL
  - NULL 연산
  - IS NULL
  - NVL
  - COALESCE
tags:
  - SQL
related:
  - "[[Aggregation_GROUP_BY]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[SQL_Constraints]]"
---
## 개념 한 줄 요약

**"공백( )도 아니고 숫자 `0`도 아닌, '아직 정의되지 않은 값(Unknown)'."** 
(마치 블랙홀 같아서, 무엇이랑 더하든 **결과는 무조건 NULL**이 되어버린다.)

---
## 가로 연산 (Horizontal Operation): 전파됨

데이터 한 줄(Row) 안에서 컬럼끼리 더하거나 뺄 때의 규칙입니다.

**규칙:** **"NULL이 하나라도 섞이면, 결과는 무조건 NULL이다."**

```sql
SELECT
    100 + 200 AS result_1,    -- 300
    100 + NULL AS result_2,   -- NULL (100이 사라짐!)
    100 * NULL AS result_3    -- NULL
FROM dual;
```

**실무 위험 상황:**
- `연봉(Salary) + 커미션(Commission)`을 계산하는데, 커미션이 `NULL`인 직원은 연봉까지 `NULL`이 되어버려 월급이 0원이 찍히는 대참사가 발생
- **해결:** 반드시 `NVL`이나 `COALESCE`로 `NULL`을 `0`으로 바꾼 뒤 연산해야 합니다.

---
## 세로 연산 (Vertical Operation): 무시됨

여러 행을 모아서 계산하는 **집계 함수(`SUM`, `AVG` 등)** 의 규칙입니다. 가로 연산과 반대입니다!

**규칙:** **"NULL인 데이터는 없는 셈 치고(무시하고) 계산한다."**

```sql
-- 데이터: [100, 200, NULL] (총 3건)

SELECT
    SUM(score) AS 합계,   -- 300 (100+200, NULL은 무시)
    AVG(score) AS 평균    -- 150 (300 / 2명) ⭐️ 중요!
FROM exams;
```

**⚠️ SQLD 함정 (평균의 배신):**
- `AVG(score)`는 NULL을 뺀 **2명**으로 나눕니다. (결과: 150)
- 만약 `NULL`을 `0`점으로 처리해서 **3명**으로 나누고 싶다면?
	- `AVG(COALESCE(score, 0))` 이렇게 써야 합니다. (결과: 100)


---
## 비교 연산 (Comparison): 알 수 없음

NULL을 찾거나 비교할 때 `{text}=`(등호)를 쓰면 망합니다.

**규칙:** **"`{sql}= NULL`은 언제나 False(Unknown)다. 반드시 `IS NULL`을 써야 한다."**

```sql
-- [틀린 문법] 아무 결과도 안 나옴
SELECT * FROM employees WHERE comm = NULL; 

-- [맞는 문법]
SELECT * FROM employees WHERE comm IS NULL;     -- NULL인 사람
SELECT * FROM employees WHERE comm IS NOT NULL; -- NULL 아닌 사람
```

**[심화] `NULL = NULL`은 참일까?**

- 아니요. "알 수 없는 값"과 "알 수 없는 값"이 같은지조차 알 수 없기 때문입니다.
- 그래서 `JOIN`을 걸 때 양쪽 컬럼이 모두 `NULL`이면 매칭되지 않습니다. (조인 실패)


**[심화] 비교 연산자의 함정 (WHERE 절의 배신)**

**"조건을 걸면 NULL 데이터는 자동으로 증발합니다."**

`WHERE col2 > 5`를 하면, 5보다 큰 값만 남는 게 아니라 **NULL인 값도 함께 사라집니다.**
(이유: `NULL > 5` 의 결과는 `Unknown`이므로 `True`가 아니어서 탈락!)

**예시 데이터 (`SAMPLE` 테이블)**

|**col1**|**col2**|**비고**|
|---|---|---|
|10|**10**|**통과 (10 > 5)**|
|20|3|탈락 (3 <= 5)|
|**30**|**NULL**|**탈락 (NULL > 5 -> Unknown)** ⭐️|

**결과 분석:**
- **3번째 행(30, NULL)** 은 `col1`에 멀쩡히 `30`이라는 값이 있지만, `WHERE` 절 조건(`col2 > 5`)을 통과하지 못해 **행 전체가 버려집니다.**
- 결국 **1번째 행(10, 10)** 만 남아서 집계됩니다.
- **결과값:** `SUM(col1) = 10`, `SUM(col2) = 10` (30은 더해지지 않음!)
	- **해결책:** NULL까지 포함해서 비교하고 싶다면 `NVL`이나 `OR` 조건을 써야 합니다.
	- `WHERE NVL(col2, 0) > 5` (0으로 바꿔서 비교)
	- `WHERE col2 > 5 OR col2 IS NULL` (NULL도 명시적으로 포함)

---
## NULL 처리 함수 (Handling Functions)

DB마다 이름이 달라서 시험에 자주 나옵니다. (기능은 똑같음: NULL이면 대체값 반환)

|**데이터베이스**|**함수명**|**사용법**|**비고**|
|---|---|---|---|
|**표준 SQL**|**`COALESCE`**|`COALESCE(col, 0)`|**가장 추천 (호환성 좋음)**. 인자 여러 개 가능.|
|**Oracle**|`NVL`|`NVL(col, 0)`|SQLD 시험 기준|
|**SQL Server**|`ISNULL`|`ISNULL(col, 0)`|MySQL의 ISNULL과 다름 주의|
|**MySQL**|`IFNULL`|`IFNULL(col, 0)`|-|

>**꿀팁:** `COALESCE(a, b, c, 0)` 처럼 쓰면, a가 NULL이면 b, b도 NULL이면 c... 순서대로 찾다가 처음으로 NULL이 아닌 값을 반환합니다.


---
## 정렬 시 NULL의 위치 (ORDER BY)

정렬할 때 NULL은 어디로 갈까요? (DB마다 다름)

**Oracle:** NULL을 **가장 큰 값**으로 취급 (오름차순 시 맨 뒤) ⭐️ _SQLD 기준_
**SQL Server:** NULL을 **가장 작은 값**으로 취급 (오름차순 시 맨 앞)

### **내 맘대로 제어하기 (`NULLS FIRST` / `LAST`)**

```sql
-- 오라클에서 NULL을 맨 앞으로 보내고 싶을 때
ORDER BY salary DESC NULLS FIRST;
```

---

## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`COUNT(*)` vs `COUNT(컬럼명)` 차이"

- **`COUNT(*)`**: NULL이고 뭐고 그냥 **줄(Row) 수**를 다 셉니다. (가장 정확한 전체 행 개수)
- **`COUNT(컬럼명)`**: 해당 컬럼이 **NULL인 행은 제외**하고 셉니다.

### ② "NOT IN의 함정" (초고난이도)

- `WHERE col NOT IN (10, 20, NULL)`을 실행하면?
- **결과는 0건입니다.** (아무것도 조회되지 않음)
- 이유: `NOT IN`은 `!=` AND `!=` 연산인데, `!= NULL`이 포함되는 순간 전체 조건이 **Unknown**이 되어버리기 때문입니다.
- **해결:** `NOT IN`을 쓸 때는 서브쿼리나 리스트에서 **반드시 NULL을 제거**해야 합니다.
