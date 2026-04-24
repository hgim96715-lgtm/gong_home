---
tags:
  - SQL_TEST
related:
  - "[[00_SQL_Challenge_DashBoard]]"
  - "[[SQL_Window_Functions]]"
source: LeetCode
difficulty:
  - Hard
---
##  해결 전략 (Code Before Think)

> 세 행 이상이 **연속** `id` 이고, 각 행의 인원수가 100명 이상인 레코드를 출력하는 솔루션을 작성하세요.
> 결과 테이블을 오름차순으로 정렬하여 `visit_date`반환 **합니다** .

1. **타겟 데이터:** (어떤 테이블에서 무엇을 뽑아야 하는가?)
    -  `Stadium` 테이블
	    - `id` : 방문 번호
	    - `people` : 방문 인원 수
	    - `visit_date` : 방문날짜
2. **조건 분석:**
	- `people >= 100` 인 행만 대상
	- `id` 값이 **연속된 숫자**여야 한다.
	- 연속된 행의 개수가 **3개 이상**이어야 한다.
	- 날짜가 하루 차이일 필요는 없고, **id만 연속**이면 된다.
3. **사용할 문법:**
	- `ROW_NUMBER()` : 행 번호 부여
	- `COUNT() OVER()` : 그룹별 개수 계산
	- `PARTITION BY` : 그룹 나누기
---
## 정답 쿼리 (Solution)

```sql
-- 여기에 작성한 쿼리를 붙여넣으세요.
WITH three_peo AS (
    SELECT id, visit_date, people,
           id - ROW_NUMBER() OVER(ORDER BY id) AS peo_th
    FROM Stadium
    WHERE people >= 100

),

count_people AS (
    SELECT id, visit_date, people,
           COUNT(*) OVER(PARTITION BY peo_th) AS cnt_people
    FROM three_peo

)

SELECT id, visit_date, people
FROM count_people
WHERE cnt_people >= 3
ORDER BY visit_date;
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
