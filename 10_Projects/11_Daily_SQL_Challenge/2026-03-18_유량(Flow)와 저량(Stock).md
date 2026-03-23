---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
  - "[[SQL_Execution_Order]]"
source: solvesql
difficulty:
  - Lv.4
---
##  해결 전략 (Code Before Think)

> ‘연도별로 새롭게 소장하게 된 작품의 수’와 같이 일정 기간 동안 측정되는 지표를 ‘유량(Flow) 지표’라고 하고, ‘누적 소장 작품 수’와 같이 특정 시점에 측정되는 지표를 ‘저량(Stock) 지표’라고 합니다. 미술관의 소장 규모를 파악하기 위해 연도별로 새롭게 소장하게 된 작품의 수와, 연도별 누적 소장 작품 수를 계산하는 쿼리를 작성해주세요.

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
   - `artworks` 테이블
	   - `acquisition_date` 소장일시

1. **조건 분석:**
   - `acquisition_date`가 **NULL인 데이터는 제외**
   - 저량(Stock)은 Flow 기반으로 누적합 계산
1. **사용할 문법:**
   - `EXTRACT(year FROM date)` → 연도 추출
   - `SUM() OVER (ORDER BY ...)` → 누적합 (Stock)
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH count_flow AS(
	SELECT EXTRACT(year FROM acquisition_date) as "Acquisition year",
	COUNT(artwork_id) as "New acquisitions this year (Flow)"
	FROM artworks
	WHERE EXTRACT(year FROM acquisition_date) IS NOT NULL
	GROUP BY "Acquisition year"
)

SELECT "Acquisition year","New acquisitions this year (Flow)",
	SUM("New acquisitions this year (Flow)")OVER(ORDER BY "Acquisition year") as "Total collection size (Stock)"
FROM count_flow
```

---
##  오답 노트 & 배운 점 (Retrospective)

-  **내가 실수한 부분:**
	- Window 함수와 GROUP BY를 동시에 잘못 사용
	- `SUM(artwork_id) OVER (...)` 를 raw 데이터에 바로 적용하려고 하지만 window 함수는 **집계 이후 단계에서 동작**!!!
    
-  **새로 알게 된 함수/꿀팁:**

---
## 더 나은 풀이가 있다면?

```sql
-- 더 나은 풀이가 있을경우 
```
