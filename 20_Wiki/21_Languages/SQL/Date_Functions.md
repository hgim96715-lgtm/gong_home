---
aliases:
  - 날짜함수
  - NOW
  - DATE_TRUNC
  - EXTRACT
  - TO_CHAR
  - INTERVAL
tags:
  - SQL
related:
  - "[[Data_Types]]"
  - "[[Window_Functions]]"
---
## 개념 한 줄 요약

**"시간을 자유자재로 다루는 타임스톤(Time Stone)."** 
현재 시간을 구하거나, 날짜에서 '월(Month)'만 빼오거나, "오늘부터 3일 뒤"를 계산하는 등 시간과 관련된 모든 처리를 담당한다.

---
## 왜 필요한가? (Why)

**문제점:**
- "2024년 1월 매출만 보고 싶은데, 데이터는 '2024-01-27 14:30:21' 처럼 초 단위까지 있다."
- "사용자의 나이를 계산해야 하는데, 생년월일만 있고 나이 컬럼이 없다."
- "보고서에 '2024-01' 형태로 찍어야 하는데, DB에는 '2024/01/01'로 되어 있다."

**해결책:**
- **`DATE_TRUNC`** 로 초 단위 시간을 '월' 단위로 잘라내서 집계한다.
- **`AGE`** 함수로 생년월일과 오늘 날짜의 차이를 구한다.
- **`TO_CHAR`** 로 날짜 모양을 예쁘게 포장한다.

---

## 실무 적용 사례 (Practical Context)

1.  **월별 통계 (Monthly Stats):** 일별 데이터를 월별로 묶어서(`GROUP BY`) 매출 추이 보기.
2.  **코호트 분석 (Cohort):** "가입 후 7일 이내에 재구매한 유저" 찾기 (날짜 더하기/빼기).
3.  **요일별 분석:** "우리 서비스는 무슨 요일에 가장 접속이 많지?" (`EXTRACT(DOW)`).

---

##  Code Core Points: 필수 함수 4대장 (PostgreSQL)

### ① 현재 시간 구하기 (Current)

```sql
-- 1. 현재 날짜 + 시간 (Timestamp)
SELECT NOW();  -- 결과: 2026-01-27 10:30:00.123456+09

-- 2. 현재 날짜만 (Date)
SELECT CURRENT_DATE; -- 결과: 2026-01-27

-- [꿀팁] 어제, 내일 구하기 (연산)
SELECT CURRENT_DATE - 1; -- 어제
SELECT CURRENT_DATE + 1; -- 내일
```

### ② 날짜 버림 (Truncate) 

엑셀의 `ROUNDDOWN`과 비슷하다. 
지정한 단위 밑으로는 싹 다 초기화(01일 00시) 시킨다.

```sql
-- '월(Month)' 단위로 자르기 (가장 많이 씀)
-- 2026-01-27 15:30 -> 2026-01-01 00:00 으로 변환
SELECT DATE_TRUNC('month', NOW());

-- '주(Week)' 단위로 자르기 (그 주의 월요일로 변환)
SELECT DATE_TRUNC('week', NOW());
```

### ③ 날짜 추출 (Extract) 

날짜에서 **숫자 하나**만 쏙 뽑아낸다.

```sql
-- 연도 뽑기
SELECT EXTRACT(YEAR FROM NOW()); -- 결과: 2026

-- 월 뽑기
SELECT EXTRACT(MONTH FROM NOW()); -- 결과: 1

-- 요일 뽑기 (DOW: Day of Week)
-- 0:일요일, 1:월요일, ... 6:토요일
SELECT EXTRACT(DOW FROM NOW());
```

### ④ 날짜 연산 (Arithmetic)

PostgreSQL은 **`INTERVAL`** 키워드가 핵심이다.

```sql
-- 1. 심플하게 일(Day) 더하기 (정수 사용)
SELECT '2026-01-27'::DATE + 3; -- 3일 뒤

-- 2. 월/년 단위 더하기 (INTERVAL 사용) ⭐️ 중요
SELECT NOW() + INTERVAL '1 month'; -- 1달 뒤
SELECT NOW() - INTERVAL '1 year 2 days'; -- 1년 2일 전

-- 3. 두 날짜 사이의 간격 (Age)
SELECT AGE('2026-01-27', '1990-05-01');
-- 결과: 35 years 8 mons 26 days
```

### ⑤ 예쁘게 보여주기 (Formatting)

```sql
-- TO_CHAR(날짜, '포맷')
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD'); -- 2026-01-27
SELECT TO_CHAR(NOW(), 'YYYY년 MM월'); -- 2026년 01월
SELECT TO_CHAR(NOW(), 'Day'); -- Tuesday (요일 이름)
```

---
## 상세 분석: `EXTRACT` vs `DATE_TRUNC` 차이점

초보자가 가장 많이 헷갈리는 부분이다. **"언제 뭘 써야 해?"**

|**함수**|**결과물(Type)**|**설명**|**사용 예시**|
|---|---|---|---|
|**`EXTRACT`**|**숫자 (Double)**|날짜에서 **부품 하나**만 떼어냄|"12월에 태어난 사람 찾기" (`WHERE month = 12`)|
|**`DATE_TRUNC`**|**날짜 (Timestamp)**|날짜를 통째로 유지하되, **나머지를 0으로 초기화**|"**월별** 매출액 집계하기" (`GROUP BY month`)|


>**💡 :** "`GROUP BY` 할 때는 99% **`DATE_TRUNC`** 를 씁니다.
> `EXTRACT(MONTH)`로 그룹핑하면, 2023년 1월과 2024년 1월이 똑같이 `1`이라서 합쳐져 버리는 대참사가 일어나거든요!"


---
## 초보자가 자주 하는 실수 (Common Mistakes)

### ① "글자('2024-01-01')에다가 숫자를 더했어요."

- `'2024-01-01' + 1` 👉 에러! (글자에 숫자 못 더함)
- **해결:** `'2024-01-01'::DATE + 1` (날짜로 캐스팅 필수)

### ② "타임존(Timezone) 때문에 날짜가 달라져요."

- 서버 시간은 UTC(영국 기준)인데 한국(KST, +9)으로 생각하고 쿼리 짜면, 오전 9시 이전 데이터는 **'어제'**로 잡힘.
- **해결:** `NOW() AT TIME ZONE 'Asia/Seoul'` 처럼 타임존을 명시하거나, `DATE` 타입으로 변환해서 사용.

### ③ "두 날짜 뺐는데 이상한 숫자가 나와요."

- `TIMESTAMP - TIMESTAMP` = **`INTERVAL`** (예: `3 days 04:00:00`)
- `DATE - DATE` = **`INTEGER`** (예: `3`)
- D-Day 계산할 거면 둘 다 `::DATE`로 맞춰서 빼는 게 정신건강에 좋음.