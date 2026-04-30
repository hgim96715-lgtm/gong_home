---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_DML_CRUD]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> `Salary` 테이블에는 직원의 `id`, `name`, `sex`, `salary` 정보가 들어 있습니다.
> `sex` 컬럼은 `'m'` 또는 `'f'` 값을 가집니다.
> 문제는 `sex` 컬럼의 값을 서로 바꾸는 것입니다.
> `'m'`이면 `'f'`로 변경
> `'f'`이면 `'m'`으로 변경
> 반드시 **하나의 UPDATE 문**으로 해결해야 합니다.
> 임시 테이블을 사용하면 안 됩니다.
> `SELECT` 문을 작성하면 안 됩니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Salary` 테이블
	    - `sex`
2. **조건 분석:**
    - `sex = 'f'`인 경우 → `'m'`으로 변경한다.
    - `sex = 'm'`인 경우 → `'f'`으로 변경한다.
3. **사용할 문법:**
    - `UPDATE`
    - `SET`
    - `CASE WHEN THEN ELSE END`
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
UPDATE Salary  
SET sex = CASE  
	WHEN sex = 'f' THEN 'm'  
	ELSE 'f'  
END;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

>`IF(조건, 참일 때 값, 거짓일 때 값)` 문법 MYSQL에서 쓸수있다. 

```sql
-- 더 나은 풀이가 있을경우 
UPDATE Salary
SET sex = IF(sex = 'f', 'm', 'f');
```
