---
aliases:
  - 쿼리 전략
  - SQL 사고법
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Execution_Order]]"
  - "[[SQL_Main_Table_Strategy]]"
---
# SQL 치트시트 — 딱 이것만

---

## ① 실행 순서

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

WHERE   집계 전 조건  (집계함수 ❌)
HAVING  집계 후 조건  (집계함수 ✅)
SELECT  별칭 만드는 곳 → WHERE/HAVING 에서 못 씀
Window  SELECT 에서 실행 → WHERE 에서 못 씀 → CTE 로 감싸기
```

---

## ② 문제 유형 판단

```
행 수가 줄어?          → GROUP BY
행 수 그대로 + 계산?   → OVER (Window 함수)
두 테이블 합치기?      → JOIN
단계별 쿼리 분리?      → WITH (CTE)
```

---

## ③ 도구 선택

```
부서별 합계/평균          GROUP BY + SUM/AVG
집계 후 필터              HAVING
원본 유지 + 계산 붙이기   OVER(PARTITION BY)
소계+총계 한번에          ROLLUP

순위 (동점 건너뜀)        RANK()
순위 (동점 연속)          DENSE_RANK()
고유 번호                 ROW_NUMBER()
그룹별 최신 1건           ROW_NUMBER() WHERE rn=1
연속 구간 묶기            year - ROW_NUMBER() + GROUP BY

이전 행 값               LAG()
다음 행 값               LEAD()
이동 평균                AVG() OVER (ROWS BETWEEN N PRECEDING AND CURRENT ROW)
누적합                   SUM() OVER (ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

NULL 대체               COALESCE / NVL
조건 분기               CASE WHEN
0나누기 방지            NULLIF(값, 0)

교집합                  INNER JOIN
왼쪽 다                 LEFT JOIN
자기 자신 비교          Self JOIN
있는지 없는지           EXISTS / NOT EXISTS
위아래 붙이기           UNION ALL
있으면 UPDATE 없으면 INSERT   ON CONFLICT / MERGE
```

---

## ④ PostgreSQL 자주 쓰는 것

```
COALESCE(값, 0)                         NULL 이면 0
LEFT(str, 2)                            앞 2글자
NOW() AT TIME ZONE 'Asia/Seoul'         현재 KST 시각
LIMIT N                                 상위 N행
ON CONFLICT DO UPDATE                   있으면 UPDATE 없으면 INSERT
ON CONFLICT DO NOTHING                  중복이면 그냥 무시
DISTINCT ON (컬럼)                      그룹별 첫 행 (ROW_NUMBER 대체)
FILTER (WHERE 조건)                     조건부 집계
SPLIT_PART(str, ' ', 2)                 공백 기준 2번째 단어
DATE_TRUNC('hour', created_at)          시간 단위 자르기
INTERVAL '1 hour'                       시간 더하기
::DATE  ::TIMESTAMP  ::NUMERIC          타입 캐스팅
```

---

## ⑤ 막혔을 때

```
SELECT 별칭을 WHERE 에서 못 씀?    → CTE 로 감싸기
Window 함수를 WHERE 에서 못 씀?    → CTE 로 감싸고 바깥 필터
JOIN 후 행이 늘었나?               → 1:N 관계 → DISTINCT 또는 조건 확인
NULL = NULL 이 False 나옴?         → IS NULL 쓰기
GROUP BY 에러?                     → SELECT 컬럼 전부 GROUP BY 에 추가
누적합이 점프?                     → ROWS 명시 (RANGE 기본값 주의)
```