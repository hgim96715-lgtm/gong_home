---
aliases:
  - DISTINCT
  - 중복제거
  - Unique
tags:
  - SQL
related:
  - "[[Aggregation_GROUP_BY]]"
  - "[[SQL_SELECT_FROM]]"
---
## 개념 한 줄 요약 ️

**"단순히 '종류'만 알고 싶으면 `DISTINCT`, 그룹별로 '계산'을 해야 하면 `GROUP BY`."**

* **`DISTINCT`:** "중복된 건 다 빼고 유니크한 값만 보여줘." (단순 조회)
* **`GROUP BY`:** "이걸 기준으로 묶어서, **개수를 세거나 평균을 구해줘.**" (집계 목적)

---
##  발견 (Palmer Penguins 예제) 

### ❌ 1. 처음 했던 고민 (GROUP BY)

"서식지 별로 펭귄 종을 출력하라"고 해서 `GROUP BY`를 떠올림.
하지만 `GROUP BY`는 보통 `COUNT(*)` 같은 집계 함수와 짝꿍이라, 단순히 이름만 출력하려고 하니 어색함.

```sql
-- 실행은 되지만, 집계 함수가 없어서 목적에 맞지 않음 (혹은 에러 발생 가능)
SELECT species, island
FROM penguins
GROUP BY species, island
ORDER BY island, species;
```

### 해결책 (DISTINCT)

집계(숫자 계산)가 필요 없고, 단순히 **"어떤 조합이 있는지(Unique Combination)"** 만 궁금할 때는 `DISTINCT`가 훨씬 직관적이고 빠릅니다.

---
## 비교 요약표

| **구분**    | **DISTINCT**             | **GROUP BY**           |
| --------- | ------------------------ | ---------------------- |
| **목적**    | **중복 제거** (단순 조회)        | **집계** (통계 산출)         |
| **사용 위치** | `SELECT` 바로 뒤            | `FROM` / `WHERE` 뒤     |
| **집계 함수** | 사용 불가 (`COUNT`, `SUM` X) | **필수 사용 권장**           |
| **속도**    | 단순히 정렬해서 중복만 뺌           | 데이터를 묶고 계산 준비를 함 (무거움) |

### Tip 

"면접관이 **'중복을 제거하려면 어떻게 하죠?'** 라고 물으면 이렇게 대답하면 완벽해. 
'단순히 유니크한 값만 필요하면 **DISTINCT**를 쓰고, 각 그룹별로 카운트나 합계를 구해야 한다면 **GROUP BY**를 씁니다.' 
오늘 펭귄 문제처럼 **'종류만 나열해라'** 는 무조건 **DISTINCT**야!"

---
## 언제 `DISTINCT`를 써야 할까? 

현업에서 **"목록(Catalog)"** 이나 **"메뉴판(Menu)"** 을 뽑아야 할 때 이 패턴을 그대로 쓰면 됩니다.

### 범용 SQL 패턴

```sql
SELECT DISTINCT [대분류_컬럼], [소분류_컬럼]
FROM [테이블_이름]
ORDER BY [대분류_컬럼], [소분류_컬럼];
```

---
## 실전 활용 시나리오

**이커머스 (상품 카테고리 추출)**

> "우리 쇼핑몰에 등록된 카테고리(대분류)와 그 안에 속한 하위 카테고리(소분류) 목록을 싹 뽑아주세요."


```sql
SELECT DISTINCT category_main, category_sub
FROM products
ORDER BY category_main, category_sub;
```

**주소 데이터 (배송 가능 지역 확인)**

> "지난달 주문 내역에서 배송이 갔던 '시/도'와 '구/군' 목록만 중복 없이 보고 싶어요."

```sql
SELECT DISTINCT city, district
FROM orders
ORDER BY city, district;
```

**인사팀 (부서 및 직급 체계)**

> "현재 회사에 존재하는 부서와, 각 부서에 어떤 직급들이 있는지 확인하고 싶습니다."

```sql
SELECT DISTINCT department, job_title
FROM employees
ORDER BY department, job_title;
```

---
###  핵심 요약 (Cheatsheet)

* **목적:** 데이터의 **"종류(Unique Combination)"** 를 파악하고 싶을 때.

* **[주의] 중복 판단의 기준은 "한 줄(Row) 전체" 입니다!**
    * `DISTINCT`는 괄호 안에 있는 컬럼들을 **"하나의 세트"** 로 묶어서 봅니다.
    * **상황 1:** `(철수, 서울)` vs `(철수, 부산)`
        * 이름은 같지만 사는 곳이 다르므로 **"다른 데이터"** 취급 -> **둘 다 출력됨.**
    * **상황 2:** `(철수, 서울)` vs `(철수, 서울)`
        * 이름도 같고 사는 곳도 같으므로 **"같은 데이터"** 취급 -> **하나만 출력됨.**
    * **결론:** 만약 `A` 컬럼의 중복을 완벽하게 없애고 싶다면, 뒤에 `B` 컬럼을 붙이면 안 됩니다! (`SELECT DISTINCT A` 만 써야 함)
