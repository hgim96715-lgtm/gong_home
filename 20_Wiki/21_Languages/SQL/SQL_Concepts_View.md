---
aliases:
  - View
  - 뷰
  - 가상테이블
  - VirtualTable
  - MaterializedView
tags:
  - SQL
related:
  - "[[SQL_SELECT_FROM]]"
  - "[[Spark_DataFrame_SQL_Intro]]"
---
##  개념 한 줄 요약 

**"실제 데이터는 없고, '쿼리(Query)'만 저장된 가상의 테이블."**

* **Table (테이블):** 데이터가 **하드디스크에 물리적으로 저장**된 진짜 창고.
* **View (뷰):** 창고를 들여다보는 **'색안경(필터)'** 또는 **'창문'**.
    * 뷰를 조회하면, 그때서야 DB가 저장해둔 쿼리를 실행해서 데이터를 가져옵니다.

---
## 이걸 왜 쓰나요? (핵심 이유 3가지) 

귀찮게 뷰를 왜 만들까요? 그냥 테이블 조회하면 되지.

### ① 보안 (Security) - "월급은 비밀이야" 

직원 테이블(`Employees`)에 `이름`, `부서`, `연봉`이 다 들어있습니다.
인턴에게 테이블 접근 권한을 주면 `연봉`까지 다 보겠죠?

* **해결:** `이름`, `부서`만 보여주는 **View**를 만들어서 인턴에게는 **View의 조회 권한**만 줍니다.
* "원본 테이블은 건드리지 않으면서 보여줄 것만 보여준다."

### ② 복잡성 숨기기 (Abstraction) - "긴 쿼리 치기 싫어" 

매번 `JOIN`을 5번씩 해서 데이터를 가져와야 한다면?
* **해결:** 복잡한 `JOIN` 쿼리를 **View**로 저장해두고, 남들은 그냥 `SELECT * FROM my_view` 한 줄만 쓰게 합니다. (스파크의 TempView 사용 이유와 동일!)

### ③ 독립성 (Independence)

나중에 원본 테이블의 컬럼 이름이 바뀌어도, View만 수정하면 View를 사용하는 프로그램들은 코드를 안 고쳐도 됩니다.

---
## 문법 (Syntax) 

아주 간단합니다. `CREATE TABLE` 대신 `CREATE VIEW`를 씁니다.

```sql
-- 1. 뷰 만들기 (저장)
-- 'high_salary_view'라는 이름으로 쿼리를 저장함
CREATE VIEW high_salary_view AS
SELECT name, department
FROM employees
WHERE salary > 10000;

-- 2. 뷰 사용하기 (조회)
-- 사용자는 이게 뷰인지 테이블인지 모름. 그냥 테이블처럼 씀.
SELECT * FROM high_salary_view;

-- 3. 뷰 삭제하기
DROP VIEW high_salary_view;
```

---
## [심화] Materialized View (M.View) 

- **일반 View (Virtual):** 조회할 때마다 매번 쿼리를 실행함. (데이터 실체 없음, 실시간성 보장)
- **Materialized View (물리 뷰):** 쿼리 실행 결과를 **진짜 테이블처럼 디스크에 저장(Caching)** 해버림.
    - **장점:** 조회 속도가 엄청 빠름 (이미 계산 끝났으니까).
    - **단점:** 원본 데이터가 바뀌면 M.View도 갱신(Refresh)해줘야 함. (데이터가 실시간이 아닐 수 있음).

---
## Table vs View vs Spark View 비교

|**구분**|**Table (테이블)**|**View (일반 뷰)**|**Spark TempView**|
|---|---|---|---|
|**데이터 저장**|**O (디스크)**|X (쿼리만 저장)|X (메모리 포인터)|
|**물리적 공간**|차지함|차지하지 않음|메모리 차지함|
|**목적**|데이터 보관|보안, 편의성|**SQL 문법 사용 편의성**|
|**지속성**|영구적|영구적 (명령어만)|**휘발성 (세션 끝나면 삭제)**|


>"우리가 스파크에서 썼던 `createTempView`는 **'SQL 문법을 쓰기 위해 잠시 만든 가상의 뷰'** 야.
>하지만 DB에서 말하는 진짜 **View**는 **'보안과 편의성을 위해 DB에 영구 박제해둔 쿼리'** 라고 생각하면 돼.
>**'바로 가기 아이콘'** 같은 거지. 아이콘(View)을 지운다고 원본 파일(Table)이 지워지진 않잖아?"


