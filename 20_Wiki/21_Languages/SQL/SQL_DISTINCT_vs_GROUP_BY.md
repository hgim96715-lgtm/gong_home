---
aliases:
  - DISTINCT
  - 중복제거
  - Unique
  - COUNT(DISTINCT 컬럼)
tags:
  - SQL
related:
  - "[[SQL_Aggregate_GROUP_BY]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[00_SQL_HomePage]]"
---
## 개념 한 줄 요약 ️

**"단순히 '종류(목록)'만 알고 싶으면 `SELECT DISTINCT`, 그룹 내에서 '서로 다른 종류의 개수'를 세려면 `COUNT(DISTINCT)`."**

* **`DISTINCT`:** "중복된 건 다 빼고 유니크한 값만 보여줘." (단순 조회)
* **`GROUP BY`:** "이걸 기준으로 묶어서, **개수를 세거나 평균을 구해줘.**" (집계 목적)
* **`COUNT(DISTINCT 컬럼)`:** "그룹으로 묶은 것 중에서, **중복을 제외한 진짜 종류가 몇 개인지** 세어줘."

---
##  발견 (Palmer Penguins 예제) 

### ❌ 처음 했던 고민 (GROUP BY의 남용)

"서식지 별로 펭귄 종을 출력하라"고 해서 `GROUP BY`를 떠올림.
하지만 `GROUP BY`는 보통 `COUNT(*)` 같은 집계 함수와 짝꿍이라, 단순히 이름만 출력하려고 하니 어색함.

```sql
-- 실행은 되지만, 집계 함수가 없어서 목적에 맞지 않음 (혹은 에러 발생 가능)
SELECT species, island
FROM penguins
GROUP BY species, island
ORDER BY island, species;
```

### 목록 추출의 해결책 (SELECT DISTINCT)

집계(숫자 계산)가 필요 없고, 단순히 **"어떤 조합이 있는지(Unique Combination)"** 만 궁금할 때는 `DISTINCT`가 훨씬 직관적이고 빠릅니다.

```sql
SELECT DISTINCT species, island
FROM penguins
ORDER BY island, species;
```

---
## COUNT DISTINCT

> **"`DISTINCT`는 단순히 목록을 뽑을 때도 쓰지만, 집계 함수(`COUNT`) 안에 쏙 들어갈 때 진짜 파괴력이 살아납니다!"**

### 언제 쓰는가?

쇼핑몰에서 "오늘 하루 방문자 수"를 구한다고 해봅시다.
-  `COUNT(user_id)`: 철수가 오늘 10번 접속했으면 10명으로 잡힘. (총 접속 횟수)
- **`COUNT(DISTINCT user_id)`**: 철수가 10번 접속해도 **중복을 제거하고 '1명'으로 셈.** (순수 방문자 수, DAU)

### 코딩 테스트/실무 단골 패턴 (HAVING + COUNT DISTINCT)

문제에서 **"몇 개 이상의 '서로 다른(다양한)' 값을 가진 데이터를 찾아라"** 라고 나오면 무조건 이 패턴입니다.

**예제:** "서로 다른 종류의 펭귄 종(species)을 2개 이상 보유한 섬(island)만 찾아라!"

```sql
SELECT island, COUNT(DISTINCT species) AS species_cnt
FROM penguins
GROUP BY island
HAVING COUNT(DISTINCT species) >= 2; -- 👈 "서로 다른 종이 2개 이상"
```

---
## DISTINCT 문법 체크리스트 (O/X)

> `DISTINCT`의 위치를 정확히 기억!!!

- ✅ **`SELECT DISTINCT col1, col2`**
	- (O) 올바른 문법. 전체 조회 결과에서 행(Row) 단위로 중복을 제거합니다.
- ✅ **`COUNT(DISTINCT col)`**
	- (O) 올바른 문법. 괄호 안의 특정 컬럼만 중복을 제거한 뒤 개수를 셉니다.
- ❌ **`SELECT col1, DISTINCT(col2)`**
	- (X) **절대 불가!** `DISTINCT`는 특정 컬럼 하나에만 함수처럼 씌울 수 없습니다. (오직 집계 함수 안에서만 저렇게 씁니다.) 
	- `SELECT` 뒤에 올 때는 무조건 `SELECT DISTINCT` 형태로 맨 앞에 한 번만 적어야 합니다.

---
## 비교 요약표 

|**구분**|**SELECT DISTINCT**|**GROUP BY**|**COUNT(DISTINCT)**|
|---|---|---|---|
|**목적**|**목록/메뉴판 만들기**|**데이터 묶기**|**고유값 개수 세기**|
|**사용 위치**|`SELECT` 바로 뒤|`FROM` / `WHERE` 뒤|`SELECT`나 `HAVING` 절 내부|
|**작동 원리**|조회된 행 전체의 중복을 가림|지정된 기준으로 데이터를 파티셔닝|그룹 내에서 중복을 뺀 개수만 셈|

---
## 실전 활용 시나리오 (목록 뽑기)

현업에서 **"목록(Catalog)"** 이나 **"메뉴판(Menu)"** 을 뽑아야 할 때는 `SELECT DISTINCT`를 씁니다.

### **이커머스 (상품 카테고리 추출)**

>"우리 쇼핑몰에 등록된 대분류/소분류 카테고리 목록만 중복 없이 뽑아주세요."

```sql
SELECT DISTINCT category_main, category_sub
FROM products
ORDER BY category_main, category_sub;
```

### 주소 데이터 (배송 가능 지역 확인)

> "지난달 주문 내역에서 배송이 갔던 '시/도'와 '구/군' 목록만 중복 없이 보고 싶어요."

```sql
SELECT DISTINCT city, district
FROM orders
ORDER BY city, district;
```

---
## ⚠️ 핵심 주의사항 (중복 판단의 기준)

`SELECT DISTINCT`를 쓸 때 중복 판단의 기준은 **"조회하는 한 줄(Row) 전체"** 입니다.

- `DISTINCT`는 뒤에 적힌 컬럼들을 **"하나의 세트"** 로 묶어서 봅니다.
	- **상황 1:** `(철수, 서울)` vs `(철수, 부산)` > 이름은 같지만 사는 곳이 다르므로 **"다른 데이터"** 취급 -> **둘 다 출력됨.**
	- **상황 2:** `(철수, 서울)` vs `(철수, 서울)` >이름도 같고 사는 곳도 같으므로 **"같은 데이터"** 취급 -> **하나만 출력됨.**

> **결론:** 만약 `A` 컬럼의 중복만 완벽하게 없애고 싶다면, 뒤에 `B` 컬럼을 같이 `SELECT` 하면 안 됩니다! (`SELECT DISTINCT A` 만 써야 함)