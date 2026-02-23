---
aliases:
  - SQL 데이터 병합
  - UNION과 JOIN의 차이
  - UNION
  - UNION ALL
tags:
  - SQL
related:
  - "[[00_SQL_HomePage]]"
  - "[[SQL_JOIN_Concept]]"
  - "[[SQL_CASE_WHEN]]"
---
# 위아래로 합치기(UNION) vs 옆으로 붙이기(JOIN)

## 한줄 요약

"데이터를 합치는 방향이 다릅니다. 레고 블록을 위로 높게 쌓을 것인가(UNION), 옆으로 넓게 끼울 것인가(JOIN)의 차이입니다."

---
## UNION : 위아래로 길게 쌓기 (Vertical)

엑셀에서 `1월 매출` 시트 밑에 `2월 매출` 시트의 데이터를 그대로 복사해서 **이어 붙이는 작업**과 같습니다. 데이터의 행(Row) 개수가 늘어납니다.

### 성립 조건 (매우 엄격함)

위아래로 블록을 쌓으려면 블록의 모양이 같아야 하듯, 두 테이블의 **컬럼 개수와 데이터 타입이 완벽하게 일치**해야 합니다.

**언제 사용하나요?**
과거 데이터와 현재 데이터를 합칠 때 (예: `2023_logs` + `2024_logs`)
여러 지역의 동일한 형태의 데이터를 모을 때 (예: `서울_지점_매출` + `부산_지점_매출`)

```sql
-- 두 테이블을 위아래로 합침
SELECT user_id, amount, date FROM 2023_sales
UNION ALL
SELECT user_id, amount, date FROM 2024_sales;
```

### 데이터 엔지니어 실무 꿀팁: `UNION` vs `UNION ALL`

- **`UNION`:** 합칠 때 **중복된 행을 찾아서 제거**합니다. (내부적으로 정렬 작업을 수행하므로 데이터가 많을 경우 속도가 **매우 느려집니다!**)
- **`UNION ALL` (⭐ 실무 권장):** 중복 제거 따위는 신경 쓰지 않고 **무지성으로 그냥 밑에다 가져다 붙입니다.** (속도가 훨씬 빠릅니다.)

> **"실무에서는 특별한 이유가 없다면 무조건 `UNION ALL`을 기본으로 씁니다."**

---
## 조건부 집계(`CASE WHEN`) vs `UNION ALL`

"하나의 테이블에서 여러 조건의 개수를 더해야 할 때, 코드를 어떻게 짜는 것이 가장 빠를까요?"

특정 조건(`Standard` 배송인 것 + `온라인 반품`인 것)의 개수를 합쳐서 계산할 때, 크게 두 가지 접근법

### `CASE WHEN` + 더하기(`+`)

테이블을 **딱 한 번만 쭉 읽으면서(1 Table Scan)**, 각 조건에 맞을 때마다 카운터를 올리고 마지막에 그 숫자들을 더하는 방식입니다.

```sql
SELECT 
    EXTRACT(year FROM purchased_at) as year,
    -- 조건 A의 개수와 조건 B의 개수를 각각 세서 더함
    COUNT(CASE WHEN shipping_method='Standard' THEN 1 END) +
    COUNT(CASE WHEN is_returned='true' and is_online_order='true' THEN 1 END) as standard
FROM transactions
GROUP BY EXTRACT(year FROM purchased_at)
```

- **장점:** 데이터를 한 번만 스캔하므로 **데이터가 대용량일 때 압도적으로 빠릅니다.** (Spark, BigQuery 등에서 매우 권장)
- **주의할 점 (Double Counting):** 만약 어떤 주문이 `Standard` 배송이면서 동시에 `온라인 반품`이라면? 앞에서도 1이 세어지고 뒤에서도 1이 세어져서 **중복 카운트(2번 더해짐)** 가 됩니다.

###  `UNION ALL`을 활용한 방식

똑같은 결과를 만들기 위해 데이터를 위아래로 쌓은 뒤, 다시 밖에서 그룹핑하여 더하는 방식입니다

```sql
SELECT year, SUM(standard_count) as standard
FROM (
    -- 1. Standard 배송인 것만 먼저 세기
    SELECT EXTRACT(year FROM purchased_at) as year, COUNT(*) as standard_count
    FROM transactions 
    WHERE shipping_method='Standard'
    
    UNION ALL
    
    -- 2. 온라인 반품인 것만 따로 세서 밑에 이어 붙이기
    SELECT EXTRACT(year FROM purchased_at) as year, COUNT(*) as standard_count
    FROM transactions 
    WHERE is_returned='true' and is_online_order='true'
)
GROUP BY year
```

**장점:** 각 `SELECT` 문마다 조건(`WHERE`)이 명확해서, 데이터베이스가 **인덱스(Index)** 를 아주 찰떡같이 타기 좋습니다. 
**치명적인 단점:** 원본 테이블(`transactions`)을 **두 번 읽어야 합니다.** (Multi Table Scan). 만약 데이터가 10억 건이라면, 20억 건을 뒤져야 하므로 I/O 비용이 엄청나게 증가할 수 있습니다.

### 엔지니어는 뭘 써야 할까?

1. **분석용 DB (BigQuery, Spark, Snowflake 등):** 무조건 **`1번 (CASE WHEN)`** 이 승리합니다. 대용량 데이터는 테이블을 여러 번 읽는 `UNION ALL`을 극도로 싫어합니다.
2. **트랜잭션 DB (MySQL, PostgreSQL 등):** 데이터가 적고 인덱스가 잘 걸려있다면 **`2번 (UNION ALL)`** 이 더 빠를 때도 있습니다. (각각의 인덱스를 쏙쏙 뽑아오기 때문)

---
## JOIN : 옆으로 넓게 붙이기 (Horizontal)

엑셀의 **VLOOKUP**과 똑같습니다. A 테이블에 있는 데이터를 기준으로, B 테이블에 있는 '추가 정보'를 끌어와서 옆에 열(Column)을 새로 붙이는 작업입니다.

### 성립 조건 (연결고리 필수)

두 테이블을 하나로 묶어줄 **공통된 기준점(Key, 예: `user_id`, `product_id`)** 이 반드시 있어야 합니다.


### 언제 사용하나요?

- 트랜잭션 데이터에 마스터 데이터를 붙여 **정보를 풍부하게(Enrichment) 만들 때**
- (예: `결제내역` 테이블에는 `user_id`만 있는데, 보고서를 위해 `회원정보` 테이블을 연결해서 '고객 이름'과 '나이'를 옆에 붙여야 할 때)

```sql
-- 공통 키(user_id)를 기준으로 옆으로 정보(name, age)를 붙임
SELECT 
    A.order_id, 
    A.amount, 
    B.name, 
    B.age
FROM orders A
LEFT JOIN users B 
ON A.user_id = B.user_id;
```

---
##  한눈에 보는 핵심 요약 (Cheat Sheet)

|**비교 항목**|**UNION (위아래)**|**JOIN (옆으로)**|
|---|---|---|
|**엑셀 비유**|행(Row) 복사해서 밑에 붙여넣기|VLOOKUP으로 옆에 새로운 열(Column) 가져오기|
|**데이터 변화**|데이터가 **길어짐** (행 개수 증가)|데이터가 **뚱뚱해짐** (컬럼 개수 증가)|
|**필수 조건**|두 테이블의 **컬럼 수와 타입이 같아야 함**|두 테이블을 연결할 **공통 키(Key)**가 있어야 함|
|**주요 목적**|분할된 이력/로그 데이터를 하나로 통폐합|부족한 속성(이름, 등급 등)을 다른 테이블에서 보강|

