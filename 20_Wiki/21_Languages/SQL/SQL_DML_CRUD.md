---
aliases:
  - 데이터 조작
  - INSERT
  - UPDATE
  - DELETE
  - CRUD
tags:
  - SQL
related:
  - "[[SQL_DDL_Create]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
  - "[[Python_Database_Connect]]"
---
## 개념 한 줄 요약

**"지어진 집에 데이터를 실제로 넣고(`INSERT`), 고치고(`UPDATE`), 지우는(`DELETE`) 이사/인테리어 작업"**

---
## 데이터 넣기 (`INSERT`)

- 문법: `{sql}INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);`

```sql
-- 문법: INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);

-- 예시
INSERT INTO user_logs (user_name, price) 
VALUES ('김철수', 5000);

-- 꿀팁: 여러 줄 한 번에 넣기 (Bulk Insert)
INSERT INTO user_logs (user_name, price) 
VALUES 
    ('이영희', 3000),
    ('박민수', 15000);
```

## python에서 INSERT 

실무에서는 SQL을 직접 타이핑하는 게 아니라 **Python 코드에서 데이터를 밀어넣는** 방식을 많이 씁니다.

```python
cursor.execute("""
    INSERT INTO user_logs (user_name, price, timestamp)
    VALUES (%s, %s, NOW())
""", (data["user_name"], data["price"]))
```

>`NOW()`는 SQL 안에 쓰니까 파이썬 튜플에서 빠지는 게 포인트!


---
## 데이터 수정하기 (`UPDATE`) 

**[초보자 주의 1순위]** 🚨 반드시 **`WHERE` (조건)** 를 붙여야 합니다. 
안 그러면 **모든 데이터가 다 바뀝니다.**

```sql
-- ❌ 절대 금지 (모든 유저 이름이 'Unknown'으로 바뀜 대참사)
UPDATE user_logs SET user_name = 'Unknown';

-- ✅ 올바른 예 (ID가 'user_123'인 사람만 바꿈)
UPDATE user_logs 
SET price = 6000 
WHERE user_id = 'user_123';
```

---
## 데이터 삭제하기 (`DELETE`) 

**[초보자 주의 2순위]** 🚨 이것도 **`WHERE`** 없이 쓰면 테이블이 텅 빕니다.

```sql
-- ❌ 절대 금지 (데이터 전멸)
DELETE FROM user_logs;

-- ✅ 올바른 예 (가격이 0원인 로그만 삭제)
DELETE FROM user_logs 
WHERE price = 0;
```


**비교:** `TRUNCATE`는 조건 없이 싹 지우는 거고, `DELETE`는 `WHERE`로 골라서 지울 수 있습니다.