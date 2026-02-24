---
aliases:
  - 테이블생성
  - DDL
  - CREATE
  - DROP
  - 구조변경
tags:
  - Python
related:
  - "[[Python_Database_Connect]]"
  - "[[SQL_Data_Types]]"
  - "[[SQL_Constraints]]"
  - "[[00_SQL_HomePage]]"
  - "[[SQL_DML_CRUD]]"
---
## 개념 한 줄 요약

**"데이터가 들어갈 '집(Table)'을 짓고(`CREATE`), 부수고(`DROP`), 수리하는(`ALTER`) 건축 작업"**

---
## 테이블 만들기 (`CREATE TABLE`) 

가장 기본이 되는 명령어입니다. 
컬럼 이름과 **자료형(Type)** 을 반드시 정해줘야 합니다.

문법: `{sql}CREATE TABLE 테이블이름 ( 컬럼명 자료형 [제약조건], ... );`

```sql
-- 문법: CREATE TABLE 테이블이름 ( 컬럼명 자료형 [제약조건], ... );

CREATE TABLE user_logs (
    event_uuid VARCHAR(50) PRIMARY KEY,  -- 문지기(PK) 설정
    user_name VARCHAR(50),               -- 문자열
    price INT,                           -- 숫자
    created_at TIMESTAMP DEFAULT NOW()   -- 날짜 (기본값 현재시간)
);
```

---
## 테이블 완전히 삭제하기 (`DROP TABLE`) 

**[주의]** 테이블 자체를 폭파합니다. 데이터만 지우는 게 아니라 **테이블 구조까지 흔적도 없이 사라집니다.** 
복구가 불가능하니 실무에서는 정말 신중해야 합니다.

```sql
-- 만약 존재한다면 삭제해라 (에러 방지용 안전 장치)
DROP TABLE IF EXISTS user_logs;
```

---
## 데이터만 싹 비우기 (`TRUNCATE`) 

테이블(집)은 남겨두고, 안의 가구(데이터)만 싹 갖다 버립니다.

-  **장점:** `DELETE`보다 속도가 엄청나게 빠릅니다.
- **용도:** 테스트 데이터를 초기화할 때 많이 씁니다.

```sql
TRUNCATE TABLE user_logs;
```

 **`TRUNCATE` vs `DELETE` 차이**

|           | `DELETE` | `TRUNCATE` |
| --------- | -------- | ---------- |
| 속도        | 느림       | 빠름         |
| 조건(WHERE) | 가능       | 불가능        |
| 롤백(취소)    | 가능       | DB마다 다름    |
| 용도        | 일부 삭제    | 전체 초기화     |

---
## 구조 변경하기 (`ALTER TABLE`) 

이미 지어진 집에 방을 추가하거나 이름을 바꾸는 것 말고도, **벽지를 바꾸거나(타입 변경), 잠금장치를 다는(제약조건)** 작업도 가능합니다.

### ① 자료형(Type) 변경하기 (`TYPE`) 

"아차! `price`를 문자로 만들었네? 숫자로 바꿔야지!" 할 때 씁니다.

- 문법:`{sql} ALTER COLUMN 컬럼명 TYPE 새로운자료형`

```sql
-- 문법: ALTER COLUMN 컬럼명 TYPE 새로운자료형
ALTER TABLE user_logs 
ALTER COLUMN price TYPE INT;

-- (PostgreSQL 전용 꿀팁) 
-- 문자를 숫자로 억지로 바꿀 때, 'USING'을 써서 변환법을 알려줘야 에러가 안 남
ALTER TABLE user_logs 
ALTER COLUMN price TYPE INT USING price::integer;
```

### ② 필수 입력 여부 바꾸기 (`NOT NULL`) 

"원래는 이름 안 적어도 됐는데, 이제부턴 무조건 적게 하자!"

```sql
-- 1. 필수 입력으로 변경 (빈칸 금지)
ALTER TABLE user_logs 
ALTER COLUMN user_name SET NOT NULL;

-- 2. 다시 선택 입력으로 변경 (빈칸 허용)
ALTER TABLE user_logs 
ALTER COLUMN user_name DROP NOT NULL;
```

### ③ 기본값 설정하기 (`DEFAULT`) 

"데이터 넣을 때 날짜 안 적으면, 자동으로 오늘 날짜 들어가게 하자!"

```sql
-- 1. 기본값 설정 (입력 안 하면 NOW() 들어감)
ALTER TABLE user_logs 
ALTER COLUMN created_at SET DEFAULT NOW();

-- 2. 기본값 삭제 (이제부턴 자동 입력 안 됨)
ALTER TABLE user_logs 
ALTER COLUMN created_at DROP DEFAULT;
```

---
## `ALTER TABLE` 요약표 (Cheatsheet)

|**작업**|**명령어 키워드**|**PostgreSQL 예시**|
|---|---|---|
|**컬럼 추가**|`ADD COLUMN`|`ADD COLUMN email VARCHAR(100)`|
|**이름 변경**|`RENAME COLUMN`|`RENAME COLUMN price TO cost`|
|**컬럼 삭제**|`DROP COLUMN`|`DROP COLUMN email`|
|**타입 변경**|`TYPE`|`ALTER COLUMN price TYPE INT`|
|**필수 설정**|`SET NOT NULL`|`ALTER COLUMN name SET NOT NULL`|
|**기본값**|`SET DEFAULT`|`ALTER COLUMN reg_date SET DEFAULT NOW()`|
>**주의:** 테이블에 데이터가 10억 개쯤 있을 때 `ALTER`를 잘못 날리면, DB가 그 10억 개를 고치느라 **몇 시간 동안 멈출(Lock)** 수도 있습니다. 사람이 없는 새벽 시간에 하는 것이 원칙입니다!