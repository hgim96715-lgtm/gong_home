---
aliases:
  - SQL 실행순서
  - 작성순서
  - Parsing Order
  - Query Order
  - SQL 작동원리
tags:
  - SQL
related:
  - "[[HAVING_vs_WHERE]]"
  - "[[SQL_SELECT_FROM]]"
  - "[[01_SQL_Thinking_Roadmap]]"
  - "[[00_SQL_HomePage]]"
---
## 개념 한 줄 요약

우리가 코드를 치는 순서(**Writing Order**)와 데이터베이스가 실제로 일을 처리하는 순서(**Execution Order**)는 **완전히 정반대**야. 
이 괴리 때문에 수많은 에러가 발생해.

---
##  The Cheat Sheet (한눈에 보기) 

이 표는 머릿속에 박제해둬야 해.

| 순서 | **작성 순서 (Writing)** ✍️ | **실행 순서 (Execution)** ⚙️ | 핵심 포인트 (Why?) |
| :--- | :--- | :--- | :--- |
| **1** | `SELECT` | **`FROM` / `JOIN`** | 일단 재료(테이블)를 가져와야 요리를 하지! |
| **2** | `FROM` | **`WHERE`** | 가져온 재료 중 썩은 건 버려 (필터링 1) |
| **3** | `WHERE` | **`GROUP BY`** | 남은 재료를 용도별로 묶어 (팀 나누기) |
| **4** | `GROUP BY` | **`HAVING`** | 묶은 팀 중에서 기준 미달 탈락시켜 (필터링 2) |
| **5** | `HAVING` | **`SELECT`** | 이제야 결과물(컬럼)을 접시에 담아 |
| **6** | `ORDER BY` | **`ORDER BY`** | 접시를 예쁘게 나열해 |
| **7** | `LIMIT` | **`LIMIT`** | 딱 정해진 개수만 서빙해 |

---
## Detailed Breakdown (단계별 상세)

DB 엔진이 되어 데이터를 처리한다고 상상해봐.

### Step 1. `FROM` & `JOIN` (데이터셋 확보)

* **"어디서 가져올까?"**
* 테이블을 메모리에 올리고, 조인이 있다면 두 테이블을 붙여서 가상의 큰 테이블을 만들어.
* 🚨 **주의:** 아직 별칭(Alias)이나 컬럼 계산이 안 된 상태야.

### Step 2. `WHERE` (1차 필터링 - Row)

* **"누구를 살려둘까?"**
* 조건에 맞지 않는 행(Row)을 가차 없이 버려.
* 🚨 **에러 포인트:** `SELECT`에서 만든 별칭(`AS total`)을 여기서 쓰면 에러 남! (아직 `SELECT` 실행 전이라서 DB는 `total`이 뭔지 모름)

### Step 3. `GROUP BY` (그룹핑)

* **"어떻게 묶을까?"**
* 살아남은 데이터를 기준 컬럼으로 묶어서 하나의 '그룹'으로 만들어.

### Step 4. `HAVING` (2차 필터링 - Group)

* **"어떤 그룹을 살려둘까?"**
* 묶여진 그룹의 통계(집계함수 결과)를 보고 필터링해.

### Step 5. `SELECT` (데이터 추출)

* **"무엇을 보여줄까?"**
* 이제야 우리가 원하던 컬럼을 뽑고, 함수(`SUM`, `COUNT`)를 계산하고, 별칭(`AS`)을 붙여.

### Step 6. `ORDER BY` (정렬)

* **"어떻게 나열할까?"**
* 뽑힌 데이터를 순서대로 줄 세워.
* ✅ **가능:** 여기서는 `SELECT`에서 만든 **별칭(Alias)을 쓸 수 있어!** (이미 `SELECT`가 끝났으니까)

### Step 7. `LIMIT` (건수 제한)

* **"몇 개만 줄까?"**
* 위에서부터 N개만 뚝 잘라서 사용자에게 던져줘.

---
##  Common Pitfalls (자주 틀리는 이유) 

### Q. "왜 `WHERE` 절에서 별칭(Alias)을 못 써요?"

```sql
SELECT amount * 100 AS total_amount  -- 4. 여기서 별명을 지었는데
FROM orders                          -- 1. 가져오고
WHERE total_amount > 5000;           -- 2. 여기서 쓰려니까 에러! (Unknown column)
```

- **이유:** `WHERE`는 2번 단계고, `SELECT`는 5번 단계라서. 2번 시점에는 `total_amount`라는 이름이 아직 세상에 없어.

### Q. "ORDER BY에서는 별칭 써도 되나요?"

- **정답:** **YES!** ⭕️
- **이유:** `ORDER BY`는 6번 단계라서, 5번(`SELECT`)에서 만든 별칭을 알아볼 수 있어.