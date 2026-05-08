---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Date_Functions]]"
source: LeetCode
difficulty:
  - Easy
---
##  해결 전략 (Code Before Think)

> `2019-07-27`을 포함하여 과거 30일 동안의 일일 활성 사용자 수를 구하는 문제입니다.
> 사용자가 특정 날짜에 최소 한 번 이상 활동했다면, 그 날짜의 활성 사용자로 간주합니다.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Activity` 테이블
	    - `activity_date`
	    - `user_id`
2. **조건 분석:**
    - `2019-07-27 - 30일`은 `2019-06-27`이지만, 이 날짜까지 포함하면 총 31일이 되므로 제외
    - 같은 사용자가 같은 날 여러 번 활동할 수 있으므로 `COUNT(user_id)`가 아니라 `COUNT(DISTINCT user_id)`를 사용
    - 날짜별 집계를 해야 하므로 `GROUP BY activity_date`가 필요
3. **사용할 문법:**
    - `COUNT(DISTINCT user_id)`
    - `GROUP BY activity_date`
    - MySQL `DATE_SUB('2019-07-27', INTERVAL 30 DAY)`
    - PostgreSQL `DATE '2019-07-27'`- `INTERVAL '30 days'`
---
## 정답 쿼리 (Solution)

```sql
-- PostgreSQL
SELECT 
    activity_date AS day,
    COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date > DATE '2019-07-27' - INTERVAL '30 days'
  AND activity_date <= DATE '2019-07-27'
GROUP BY activity_date;
```

```sql
-- MySQL
SELECT 
    activity_date AS day,
    COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date > DATE_SUB('2019-07-27', INTERVAL 30 DAY)
  AND activity_date <= '2019-07-27'
GROUP BY activity_date;
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    - 처음에는 `BETWEEN`을 사용해서 날짜 범위를 다음처럼 작성했다.
    - BETWEEN DATE '2019-07-27' - INTERVAL '30 days' AND DATE '2019-07-27'
    - 문제에서 원하는 것은 `2019-07-27`을 포함한 30일이므로 실제 시작일은 `2019-06-28`이어야 한다.
-  **새로 알게 된 함수/꿀팁:**
	- PostgreSQL에서 날짜 계산은 `DATE '날짜' - INTERVAL '기간'` 형식
	- MySQL에서 날짜 계산은 `DATE_SUB(날짜, INTERVAL 숫자 DAY)` 형식

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우
SELECT 
    activity_date AS day,
    COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date BETWEEN DATE '2019-07-27' - INTERVAL '29 days'
                        AND DATE '2019-07-27'
GROUP BY activity_date;
```

