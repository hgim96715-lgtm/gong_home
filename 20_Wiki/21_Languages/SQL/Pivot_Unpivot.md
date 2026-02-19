---
aliases:
  - SQL 피벗
  - SQL 언피벗
  - 행열변환
  - UNION ALL
  - CROSS JOIN
tags:
  - SQL
related:
  - "[[SQL_CASE_WHEN]]"
  - "[[Aggregation_GROUP_BY]]"
  - "[[00_SQL_HomePage]]"
---
## 개념 한 줄 요약

**"데이터의 모양(Shape)을 90도로 회전시키는 기술."**

- **Pivot (피벗):** 세로로 긴 데이터(Long) ➔ 가로로 넓은 데이터(Wide)로 변환. (사람이 보기 좋음)
- **Unpivot (언피벗):** 가로로 넓은 데이터(Wide) ➔ 세로로 긴 데이터(Long)로 변환. (기계가 계산하기 좋음)

---
## 왜 필요한가? (Why)

**1. Pivot (행 ➔ 열): 보고서 만들기**

- **상황:** DB에는 `[2023, 100원]`, `[2024, 200원]` 처럼 세로로 저장되어 있다.
- **문제:** 팀장님이 "2023년이랑 2024년 매출 **나란히(옆으로)** 놓고 비교해봐."라고 한다.
- **해결:** 연도를 컬럼으로 끌어올린다.

**2. Unpivot (열 ➔ 행): 데이터 정규화 (Normalization)**

- **상황:** 엑셀로 관리하던 `[이름, 국어점수, 영어점수, 수학점수]` 테이블을 DB에 넣었다.
- **문제:** "전체 과목 평균 점수를 구해줘." -> `AVG(국어 + 영어 + 수학)` 이렇게 하려니 쿼리가 너무 더럽다. 과목이 늘어나면 쿼리를 또 고쳐야 한다.
- **해결:** 과목 컬럼들을 `[이름, 과목명, 점수]` 형태의 세로 데이터로 녹여내야 `GROUP BY`가 가능하다.

---
## 실무 적용 사례 (Practical Context)

1. **Pivot:** 월별 매출 리포트 (1월, 2월, 3월... 컬럼 만들기)
2. **Unpivot:** 설문조사 데이터 (문항1, 문항2, 문항3 컬럼)를 분석하기 좋게 `[응답자, 문항번호, 답변]` 형태로 변환. 
3. **로그 분석:** JSON에 들어있던 키-값 쌍을 풀어서 행으로 만들 때.

---
## Code Core Points: ① Pivot (행 ➔ 열)

**핵심 공식:** `GROUP BY` + `MAX(CASE WHEN ...)`

> "여러 행을 **하나로 압축(GROUP BY)** 하면서, 조건에 맞는 것만 **골라내서(CASE WHEN)** 컬럼으로 만든다."

```sql
-- 원본: [연도, 매출] 세로 데이터
SELECT
    product_id,
    -- 2023년 데이터만 건져올림
    MAX(CASE WHEN year = 2023 THEN amount END) AS sales_2023,
    -- 2024년 데이터만 건져올림
    MAX(CASE WHEN year = 2024 THEN amount END) AS sales_2024
FROM sales_table
GROUP BY product_id;
```

(참고: `SUM`을 써도 되고, 값이 하나뿐이면 `MAX`를 써도 됩니다.)

---
## Code Core Points: ② Unpivot (열 ➔ 행) ⭐️

**핵심 공식:** `UNION ALL` (가장 직관적이고 표준적인 방법)

> "가로로 되어 있는 컬럼들을 가위로 잘라서, **세로로 풀칠해 이어 붙인다(UNION ALL)**."

**상황:** `scores` 테이블에 `math`, `eng` 컬럼이 있다. 이걸 `subject`, `score`로 바꾸고 싶다.

```sql
SELECT student_name, 'Math' AS subject, math AS score
FROM scores

UNION ALL  -- 위아래로 이어 붙이기

SELECT student_name, 'English' AS subject, eng AS score
FROM scores;
```

결과 변환:

|**(원본)**|**Math**|**Eng**|
|---|---|---|
|철수|90|80|

**Unpivot (UNION ALL)**

|**이름**|**과목**|**점수**|
|---|---|---|
|철수|Math|90|
|철수|English|80|

---
## Unpivot (PostgreSQL / BigQuery)

`UNION ALL`은 테이블을 여러 번 읽어야 해서 성능이 안 좋습니다. 
최신 DB들은 **배열(Array)** 이나 **전용 함수**를 사용해 한 번에 처리합니다.

### PostgreSQL (`CROSS JOIN` + `LATERAL` / `VALUES`)

"가상의 테이블을 옆에 붙여서 뻥튀기하는 원리"입니다.

```sql
SELECT
    t.student_name,
    u.subject,
    u.score
FROM scores t
CROSS JOIN LATERAL (
    -- 가상의 테이블(u)을 즉석에서 만듦
    VALUES 
        ('Math', t.math),
        ('English', t.eng),
        ('Science', t.sci)
) AS u(subject, score);
```

### BigQuery (`UNPIVOT` 연산자)

빅쿼리는 엑셀처럼 `UNPIVOT`이라는 명령어를 직접 지원합니다. (가장 쉬움)

```sql
SELECT *
FROM scores
UNPIVOT (
    score FOR subject IN (math, eng, sci)
)
```

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "Pivot할 때 컬럼이 자동으로 늘어나게 할 수 없나요?"

- **질문:** "2023, 2024... 년도가 계속 생기는데 매번 `CASE WHEN`을 추가해야 하나요?"
- **답변:** **네, 표준 SQL에서는 불가능합니다.** (Dynamic SQL이라고 부름)
- SQL은 실행 전에 컬럼 개수가 정해져 있어야 합니다. (Python Pandas의 `pivot_table`은 가능하지만 SQL은 안 됨).
- 그래서 실무에서는 **Unpivot 상태(세로 데이터)로 저장**해두고, 시각화 도구(Tableau, Superset)에서 피벗을 합니다.

### ② "`UNION` vs `UNION ALL` 차이가 뭔가요?"

- **`UNION`**: 합친 다음에 **중복을 제거(Unique)** 하려고 정렬을 한 번 합니다. (느림)
- **`UNION ALL`**: 무지성으로 그냥 갖다 붙입니다. (빠름)
- Unpivot을 할 때는 어차피 과목명이 달라서 중복이 안 생기므로, **무조건 `UNION ALL`** 을 써야 성능이 좋습니다.

### ③ "Unpivot은 언제 쓰나요?"

- 주로 **데이터 전처리(ETL)** 단계에서 씁니다.
- "사용자 로그가 `login_dt`, `logout_dt`, `signup_dt`로 컬럼별로 흩어져 있는데, 이걸 시간순으로 정렬하고 싶어요."
- ➔ Unpivot으로 `event_type`(login/logout/signup)과 `event_time`으로 바꾸면 시간순 정렬(`ORDER BY event_time`)이 가능해집니다.