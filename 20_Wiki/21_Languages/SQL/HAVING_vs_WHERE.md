---
aliases:
  - WHERE vs HAVING
  - 필터링 차이
  - SQL 실행순서
  - 집계함수 필터링
tags:
  - SQL
  - Database
related:
  - "[[SQL_Filtering_WHERE]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
---
## 초보자가 자주 착각하는 포인트

**WHERE**는 그룹핑하기 **전**에 원본 데이터(Raw Data)를 거르는 **"입구 컷"** 이고,
**HAVING**은 그룹핑하고 **난 후**에 집계된 결과(Aggregated Data)를 거르는 **"출구 컷"** 이다.

---
## Why is it needed? (왜 구분할까?)

SQL 엔진 입장에서는 순서가 명확해야 해.
* "평균 점수가 80점 이상인 반을 뽑아줘"라고 했을 때,
* **WHERE**는 아직 "평균"이 계산되기 전 단계라서 무슨 말인지 못 알아들어.
* 그래서 **"계산이 끝난 후"** 에 거를 수 있는 별도의 단계(**HAVING**)가 필요한 거야.

---
##  Practical Context (실무 비교)

| 구분 | **WHERE** (입구 컷) 🚪 | **HAVING** (출구 컷) 🏆 |
| :--- | :--- | :--- |
| **실행 시점** | `GROUP BY` **하기 전** | `GROUP BY` **하고 난 후** |
| **필터 대상** | 개별 행 (Row) | 그룹화된 결과 (Group) |
| **집계 함수** | **사용 불가** ❌ (`SUM`, `COUNT` 못 씀) | **사용 필수** ⭕️ (`SUM`, `AVG` 등) |
| **예시** | "서울 사는 학생만 추려내" | "평균 90점 이상인 반만 추려내" |

---
## Code Core Points (문법 위치)

코드를 짤 때 이 순서를 절대 어기면 안 돼.

```sql
SELECT
    department,
    AVG(salary)
FROM employees

-- 1. WHERE: 묶기 전에 거른다 (입사일 2024년 이후인 사람만!)
WHERE hire_date >= '2024-01-01'

-- 2. GROUP BY: 이제 부서별로 뭉친다
GROUP BY department

-- 3. HAVING: 뭉친 결과를 보고 거른다 (평균 연봉 5천 이상인 부서만!)
HAVING AVG(salary) >= 5000;
```

---
## Detailed Analysis (실행 순서 시각화) 

SQL이 데이터를 처리하는 파이프라인을 머릿속에 그려야 해.

1. **FROM:** 테이블을 가져온다.
2. **WHERE:** 조건에 안 맞는 행(Row)은 **집계에 참여도 못하고 탈락**시킨다. (데이터 양을 줄여줌 📉)
3. **GROUP BY:** 살아남은 애들끼리 팀을 짠다.
4. **HAVING:** 팀별 성적표(통계)를 보고, 기준 미달인 팀을 탈락시킨다.
5. **SELECT:** 최종 결과를 보여준다.

---
## Performance & Optimization (성능 최적화 팁)

**"가능하면 WHERE에서 먼저 걸러라!"**

똑같은 결과를 낼 수 있다면, `HAVING`보다 `WHERE`가 훨씬 빨라.

- **Bad 👎:** 100만 명을 다 그룹핑해서 통계 낸 다음(`HAVING`), 남자만 남긴다.
- **Good 👍:** 애초에 남자만 추려서(`WHERE`), 그 사람들만 그룹핑한다. (계산할 양이 확 줄어듦!)

```sql
-- ❌ 비효율적인 쿼리
SELECT department, SUM(salary)
FROM employees
GROUP BY department
HAVING department = 'Sales'; -- 그룹 다 만들고 나서 Sales만 남김 (낭비)

-- ⭕️ 효율적인 쿼리
SELECT department, SUM(salary)
FROM employees
WHERE department = 'Sales'   -- 처음부터 Sales만 뽑아서 집계 (빠름)
GROUP BY department;
```

---
## 초보자가 자주 착각하는 포인트

**Q: "HAVING 절에는 집계함수만 올 수 있나요?"**

- A: 아니, `GROUP BY`에 쓴 컬럼도 올 수 있어.
- 하지만 `GROUP BY`에 쓴 일반 컬럼 조건은 **WHERE절에 적는 게 성능상 훨씬 이득**이야. (위의 팁 참고)

**Q: "WHERE 절에서 `AS`로 만든 별칭(Alias)을 못 써요."**

- A: 당연해. `WHERE`는 `SELECT`보다 **먼저** 실행되니까, 아직 별칭이 만들어지기 전이거든!

