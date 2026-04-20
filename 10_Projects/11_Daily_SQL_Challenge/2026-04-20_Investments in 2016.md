---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_SubQuery]]"
source: LeetCode
difficulty:
  - Medium
---
##  해결 전략 (Code Before Think)

> `Insurance` 테이블에서 다음 조건을 모두 만족하는 보험 가입자들의  **2016년 투자 금액(tiv_2016)의 합**을 구하라.
>   tiv_2015 값이 다른 사람과 같은 경우 (중복값)
>    **(lat, lon) 위치가 유일한 경우 (혼자만 있는 좌표)**
>    결과는 **소수점 둘째 자리까지 반올림**

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    - `Insurance` 테이블
		- `tiv_2015`
		- `tiv_2016`
		- `lat`
		- `lon`
2. **조건 분석:**
	-  `tiv_2015`가 **2개 이상 존재하는 값인지 확인**
	- `(lat, lon)`이 **해당 값이 1개만 존재하는지 확인**
3. **사용할 문법:**
	- `GROUP BY + HAVING` → 중복/유일 조건 처리
	-  `SUM`, `ROUND` → 최종 결과 계산
---
## 정답 쿼리 (Solution)

```sql
-- PostgreSQL
SELECT ROUND(SUM(tiv_2016)::NUMERIC, 2) AS tiv_2016
FROM Insurance
WHERE tiv_2015 IN (
    SELECT tiv_2015
    FROM Insurance
    GROUP BY tiv_2015
    HAVING COUNT(tiv_2015) > 1
)
AND (lat, lon) IN (
    SELECT lat, lon
    FROM Insurance
    GROUP BY lat, lon
    HAVING COUNT(*) = 1
);
```

```sql
-- MYSQL
SELECT ROUND(SUM(tiv_2016), 2) AS tiv_2016
FROM Insurance
WHERE tiv_2015 IN (
    SELECT tiv_2015
    FROM Insurance
    GROUP BY tiv_2015
    HAVING COUNT(tiv_2015) > 1
)
AND (lat, lon) IN (
    SELECT lat, lon
    FROM Insurance
    GROUP BY lat, lon
    HAVING COUNT(*) = 1
);
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
