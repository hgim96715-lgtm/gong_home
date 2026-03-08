---
aliases:
  - 통계함수
  - 중앙값
  - PERCENTILE
  - 분산 표준편차
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_Window_Functions]]"
---
# SQL_Statistical_Functions

## 개념 한 줄 요약

> **"COUNT, SUM, AVG 로는 부족할 때. 데이터의 분포와 퍼짐을 수치로 표현하는 통계 함수들."**

---

---

# ① 중앙값 — PERCENTILE_CONT / PERCENTILE_DISC

> 평균은 극단값(이상치)에 크게 흔들린다. **중앙값(Median)** 은 데이터를 정렬했을 때 정중앙에 위치한 값으로, 이상치에 강하다.

```
데이터: 10, 20, 30, 40, 1000

평균:   220  ← 1000 때문에 대표값으로 부적절
중앙값: 30   ← 실제 데이터를 더 잘 반영
```

## PERCENTILE_CONT — 연속 중앙값 (보간)

> 중앙값이 두 값 사이에 걸리면 **두 값의 평균으로 보간**해서 반환. 실제 데이터에 없는 값이 나올 수 있다.

```sql
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY pm10) AS pm10_중앙값
FROM air_quality;
```

```
데이터: 10, 20, 30, 40  (짝수 개)
→ 중간 두 값: 20, 30
→ PERCENTILE_CONT(0.5) = (20 + 30) / 2 = 25.0  ← 실제 없는 값
```

## PERCENTILE_DISC — 이산 중앙값 (실제 값)

> 중앙값에 가장 가까운 **실제 존재하는 값** 을 반환. 반드시 데이터셋 안에 있는 값이 나온다.

```sql
SELECT
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY pm10) AS pm10_중앙값
FROM air_quality;
```

```
데이터: 10, 20, 30, 40  (짝수 개)
→ PERCENTILE_DISC(0.5) = 20  ← 실제 존재하는 값
```

## CONT vs DISC 비교

|구분|PERCENTILE_CONT|PERCENTILE_DISC|
|---|:-:|:-:|
|반환값|보간값 (없는 값 가능)|실제 존재하는 값|
|짝수 개일 때|두 중간값의 평균|두 중간값 중 작은 값|
|주 사용처|수치형 분석|범주형 또는 실값 필요 시|

## 문법 구조

```sql
PERCENTILE_CONT(분위수) WITHIN GROUP (ORDER BY 컬럼)

분위수:
0.5  → 중앙값 (50번째 백분위)
0.25 → 1사분위수 (Q1)
0.75 → 3사분위수 (Q3)
```

## GROUP BY 와 함께 쓰기

```sql
-- 지역별 pm10 중앙값
SELECT
    region,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY pm10) AS pm10_중앙값,
    AVG(pm10)                                          AS pm10_평균
FROM air_quality
GROUP BY region;
```

---

---

# ② 표준편차 / 분산 — STDDEV / VARIANCE

> 데이터가 평균에서 얼마나 **퍼져있는지** 를 수치화한다. 값이 클수록 데이터가 고르지 않고 들쑥날쑥하다는 의미.

```sql
SELECT
    STDDEV(score)   AS 표준편차,   -- 평균에서 떨어진 정도 (단위: 원본과 동일)
    VARIANCE(score) AS 분산        -- 표준편차의 제곱 (단위: 원본의 제곱)
FROM exam_scores;
```

```
표준편차가 작다 → 데이터가 평균 근처에 모여있음 (균일)
표준편차가 크다 → 데이터가 평균에서 멀리 퍼져있음 (불균일)
```

|함수|Oracle|PostgreSQL|
|---|---|---|
|표준편차 (표본)|`STDDEV`|`STDDEV`|
|표준편차 (모집단)|`STDDEV_POP`|`STDDEV_POP`|
|분산 (표본)|`VARIANCE`|`VARIANCE`|
|분산 (모집단)|`VAR_POP`|`VAR_POP`|

---

---

# ③ 상관계수 — CORR

> 두 컬럼이 **얼마나 같이 움직이는지** 를 -1 ~ 1 사이 값으로 표현한다.

```sql
SELECT
    CORR(pm10, pm25) AS 상관계수   -- pm10 과 pm25 의 상관관계
FROM air_quality;
```

```
결과 해석:
 1.0  → 완전 양의 상관 (pm10 오르면 pm25 도 오름)
 0.0  → 상관 없음
-1.0  → 완전 음의 상관 (pm10 오르면 pm25 는 내림)

일반적 기준:
|r| >= 0.7  → 강한 상관관계
|r| >= 0.4  → 중간 상관관계
|r| < 0.4   → 약한 상관관계
```

---

---

# ④ 전체 비교표

|함수|용도|한 줄 요약|
|---|---|---|
|`AVG`|평균|이상치에 취약|
|`PERCENTILE_CONT(0.5)`|연속 중앙값|이상치에 강함, 보간값|
|`PERCENTILE_DISC(0.5)`|이산 중앙값|이상치에 강함, 실제값|
|`STDDEV`|표준편차|데이터 퍼짐 정도|
|`VARIANCE`|분산|표준편차의 제곱|
|`CORR`|상관계수|두 변수의 관계|

---

---

# 실전 예시 — 종합

```sql
-- 지역별 pm10 통계 종합
SELECT
    region,
    COUNT(*)                                                AS 측정횟수,
    ROUND(AVG(pm10), 1)                                     AS 평균,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY pm10)       AS 중앙값,
    ROUND(STDDEV(pm10), 1)                                  AS 표준편차,
    MIN(pm10)                                               AS 최솟값,
    MAX(pm10)                                               AS 최댓값
FROM air_quality
GROUP BY region
ORDER BY 중앙값 DESC;
```

---
