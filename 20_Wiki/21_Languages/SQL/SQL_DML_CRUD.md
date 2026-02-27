---
aliases:
  - 데이터 조작
  - INSERT
  - UPDATE
  - DELETE
  - MERGE
  - CRUD
tags:
  - SQL
related:
  - "[[SQL_DDL_Create]]"
  - "[[SQL_Filtering_WHERE]]"
  - "[[00_SQL_HomePage]]"
  - "[[Python_Database_Connect]]"
  - "[[SQL_Keys_and_Identifiers]]"
  - "[[SQL_Database_Transactions_TCL]]"
---
# 데이터 넣고, 고치고, 지우고, 엎어치기 DML(Data Manipulation Language)

## 개념 한 줄 요약

DML(Data Manipulation Language)은 뼈대만 있는 테이블 안에 실제 데이터(행, Row)를 **생성(INSERT), 수정(UPDATE), 삭제(DELETE), 병합(MERGE)** 하는 데이터 조작 명령어다.

---
## 왜 필요한가?

개발자가 파이썬이나 자바스크립트로 게시판이나 회원가입 폼을 만들었다고 상상해 봐. 유저가 '가입하기' 버튼을 눌렀을 때 그 정보를 DB에 저장하고(INSERT), 비밀번호를 바꾸면 수정하고(UPDATE), 탈퇴하면 지워야(DELETE) 하잖아? 즉, 껍데기뿐인 DB에 생명력을 불어넣고 실제 서비스를 굴러가게 만드는 핵심 엔진이기 때문에 절대 모르면 안 돼!

---
## Practical Context

실무 데이터 엔지니어링에서는 파이프라인(ETL)을 통해 매일 수백만 건의 데이터를 쏟아붓기 때문에 `INSERT INTO ... SELECT`나 `MERGE`(Upsert) 구문을 숨 쉬듯이 쓴다. 

---
## COMMIT / ROLLBACK — DML의 생명줄

DML은 실행한다고 바로 DB에 영구 반영되는 게 아니다. **트랜잭션(Transaction)** 안에서 동작하기 때문에 반드시 `COMMIT`을 해줘야 최종 저장된다

```sql
UPDATE 사원 SET 연봉 = 9999 WHERE 사번 = 101;
-- 아직 DB에 반영 안 됨! 나만 보이는 임시 상태

COMMIT;   -- ✅ 이제 진짜 저장됨
ROLLBACK; -- ❌ 실수했다! COMMIT 전이라면 되돌리기 가능
```

### DB별 Auto Commit 동작 차이

| DB             | 기본 동작         | 설명                                        |
| -------------- | ------------- | ----------------------------------------- |
| **SQL Server** | ✅ Auto Commit | DML 실행 즉시 자동 반영. `BEGIN TRAN` 선언해야 롤백 가능  |
| **Oracle**     | ❌ 수동 Commit   | 명시적으로 `COMMIT` 해줘야 반영됨                    |
| **PostgreSQL** | ❌ 수동 Commit   | Oracle과 동일. 단, psql 클라이언트는 기본 Auto Commit |

>💡 **PostgreSQL 주의!** `psql` 터미널이나 DBeaver 같은 GUI 툴은 기본적으로 **Auto Commit 모드**로 동작한다. 실수로 `DELETE` 날렸다가 되돌리려면 이미 늦을 수 있으니, 중요한 작업 전엔 반드시 `BEGIN;` 으로 트랜잭션을 열어줘야 한다.

```sql
BEGIN;                          -- 트랜잭션 시작 (이제부터 롤백 가능)
DELETE FROM 사원 WHERE 사번 = 101;
-- 잠깐, 맞는 조건이 맞는지 SELECT로 확인하고...
ROLLBACK; -- 또는 COMMIT;
```

>[[SQL_Database_Transactions_TCL]] 참조 

---
## Code Core Points:  `INSERT` (데이터 넣기)

- 테이블에 새로운 행(Row) 추가하기

### **① 정석 — 원하는 컬럼만 지정**

```sql
-- 기본 문법
INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);

-- 예제
INSERT INTO 사원 (사번, 이름, 부서명) 
VALUES (101, '김맥북', '데이터엔지니어링팀');
```

>지정하지 않은 컬럼은 자동으로 `NULL` 처리

### ② 생략형 — 컬럼명 없이 전체 삽입

```sql
-- 기본 문법
INSERT INTO 테이블명 VALUES (값1, 값2, 값3, ...);

-- 예제
INSERT INTO 사원 
VALUES (102, '박옵시디언', '기획팀', '대리', 3000);
```

>테이블의 **모든 컬럼**에, **만들어진 순서 그대로**, **빠짐없이** 값을 채워야 함

### ③ 대량 삽입 — 다른 테이블에서 복사

```sql
-- 기본 문법
INSERT INTO 타겟테이블명 (컬럼1, 컬럼2)
SELECT 컬럼A, 컬럼B FROM 소스테이블명 WHERE 조건;

-- 예제
INSERT INTO 우수사원 (사번, 이름)
SELECT 사번, 이름 FROM 사원 WHERE 실적 > 90;
```

> `VALUES` 없이 `SELECT` 결과를 통째로 삽입

### ⚠️ 에러가 나는 2가지 상황

**상황 1 — NULL이 들어가면 안 되는 컬럼을 비웠을 때**

 ①번 방식으로 일부 컬럼만 지정하면, 나머지는 `NULL`이 들어간다.
  그런데 해당 컬럼에 `PK` 또는 `NOT NULL` 제약조건이 걸려 있다면?

>❌ Constraint Violation 에러 발생!
>필수 컬럼은 반드시 값을 명시해야 한다.


**상황 2 — ②번 생략형에서 값 개수가 안 맞을 때**

컬럼명을 생략하면 DB는 *"전체 컬럼에 순서대로 전부 넣겠다는 뜻"* 으로 해석한다.
값이 하나라도 부족하거나 많으면?

>❌ 즉시 에러 발생!
> 테이블 컬럼 수 = VALUES 안의 값 개수, 이 공식이 반드시 성립해야 한다.

#### SQLD 표기법 — 테이블 다이어그램에서 PK 구분하기

```css
┌─────────────────┐
│      COL1       │  ← 선 위쪽 단독 = PK
├─────────────────┤
│ COL2            │
│ COL3 DEFAULT 'D'│  ← 선 아래쪽 = 일반 컬럼
└─────────────────┘
```

> 가로선으로 분리되어 **위쪽에 단독으로 있는 컬럼 = PK**
> PK는 자동으로 `NOT NULL + UNIQUE` 조건을 가진다.
> 
> 📎 [[SQL_Keys_and_Identifiers#SQLD 표기법 — 테이블 다이어그램에서 PK 구분하기|PK 표기법 더 알아보기]] 참조 

#### 실전 예제

```sql
-- SAMPLE 테이블: COL1(PK), COL2, COL3 DEFAULT 'D'
INSERT INTO SAMPLE (COL2, COL3) VALUES ('S', 'A');
```

| COL1      | COL2 | COL3 |
| --------- | ---- | ---- |
| ❓ NULL 시도 | S    | A    |

> COL1을 지정하지 않음 → NULL 삽입 시도 → PK 위반 → **❌ 에러 발생!**

#### 함정 포인트 — DEFAULT는 언제 작동하나?

|상황|결과|
|---|---|
|컬럼을 아예 지정 안 함|DEFAULT 값이 자동 삽입|
|VALUES에 값을 직접 명시|DEFAULT 무시, 명시한 값 삽입|

위 예제에서 COL3에 `'A'`를 직접 넣었으므로 DEFAULT `'D'`는 무시되고 `'A'`가 들어간다.



### 한눈에 비교

|방식|컬럼 지정|VALUES|주의사항|
|---|---|---|---|
|정석|✅ 일부 지정|✅|미지정 컬럼 → NULL|
|생략형|❌ 생략|✅|개수·순서 완벽히 일치|
|대량 삽입|✅ 일부 지정|❌|SELECT 결과와 컬럼 수 일치|

## python에서 INSERT 

실무에서는 SQL을 직접 타이핑하는 게 아니라 **Python 코드에서 데이터를 밀어넣는** 방식을 많이 씁니다.

```python
cursor.execute("""
    INSERT INTO user_logs (user_name, price, timestamp)
    VALUES (%s, %s, NOW())
""", (data["user_name"], data["price"]))
```

>`NOW()`는 SQL 안에 쓰니까 파이썬 튜플에서 빠지는 게 포인트! 📎 [[Python_Database_Connect#③ `cur.execute(sql, values)` (주문하기,실행)|python-execute]] 참고

---
## `UPDATE `(데이터 고치기)

>🚨 반드시 **`WHERE`(조건)** 를 붙여야 한다. 안 그러면 **모든 행이 다 바뀐다.**

### 기본 문법 (Syntax)

```sql
UPDATE 테이블명 
SET 컬럼1 = 변경할값1, 컬럼2 = 변경할값2
WHERE 조건;
```

### 실전 예제 & 상세 분석


```sql
-- "사번이 101인 직원의 부서를 'AI팀'으로, 연봉을 5000으로 올려라!"
UPDATE 사원 
SET 
    부서명 = 'AI팀', 
    연봉 = 5000
WHERE 사번 = 101; 
-- 🚨 경고: 여기서 WHERE 절을 빼먹고 실행하면? 
-- 회사 내 모든 직원의 부서가 AI팀이 되고 연봉이 5000으로 똑같아지는 대참사가 벌어짐!
```

---
## `DELETE`  데이터 지우기

>🚨 **`WHERE`** 없이 쓰면 테이블이 텅 빈다.

### 기본 문법 (Syntax)

```sql
DELETE FROM 테이블명 
WHERE 조건;
```

### 실전 예제 & 상세 분석

```sql
-- "퇴사한 박옵시디언(사번 102)의 데이터를 지워라!"
DELETE FROM 사원 
WHERE 사번 = 102; 
-- 💡 DELETE 뒤에는 컬럼명(예: DELETE 이름 FROM...)을 적지 않아! 행(가로줄) 전체를 날려버리는 거니까.
-- 만약 WHERE 절을 빼먹으면? 테이블 안의 데이터가 싹 다 날아감.
```

>**비교:** `TRUNCATE`는 조건 없이 싹 지우는 거고, `DELETE`는 `WHERE`로 골라서 지울 수 있습니다.

---
## `MERGE` (엎어치기 / Upsert)

"데이터가 이미 있으면 UPDATE하고, 없으면 INSERT해라!" **(실무 끝판왕)** 
중복 에러(PK 제약조건 위배)를 피하면서 최신 데이터를 유지할 때 필수로 사용한다.

### [SQLD / Oracle 표준] MERGE INTO 기본 문법

오라클의 `MERGE`는 `ON` 절에서 기준 컬럼을 비교한 뒤, 그 결과에 따라 `WHEN MATCHED` (있을 때)와 `WHEN NOT MATCHED` (없을 때)로 길을 나눠서 처리한다.

```sql
MERGE INTO 타겟테이블 T
USING 소스테이블 S
ON (T.사번 = S.사번)                   -- 💡 [핵심] 두 테이블의 사번을 비교해서 매칭되는지 확인!

-- 🟢 [조건 1] 일치하는 데이터가 있다면? (이미 존재하는 사원)
WHEN MATCHED THEN                    
    UPDATE SET 
        T.부서명 = S.부서명,           -- 타겟 테이블의 부서를 소스 테이블의 새 부서로 덮어써라!
        T.연봉 = S.연봉
    -- 🚨 주의: UPDATE 절 안에는 타겟 테이블 이름(T)을 다시 명시하지 않고 바로 SET으로 시작함!

-- 🔴 [조건 2] 일치하는 데이터가 없다면? (새로 입사한 신입사원)
WHEN NOT MATCHED THEN                
    INSERT (사번, 이름, 부서명)       -- 타겟 테이블에 새로 빈칸을 만들고
    VALUES (S.사번, S.이름, S.부서명); -- 소스 테이블의 값들을 쏙쏙 집어넣어라!
    -- 🚨 주의: INSERT 뒤에 'INTO 타겟테이블'을 쓰지 않음! (이미 맨 위 MERGE INTO에서 선언했기 때문)
```

### [실무 / PostgreSQL 최신] ON CONFLICT 기본 문법

PostgreSQL은 오라클처럼 길게 쓰지 않고, 일단 무대포로 `INSERT`를 시도한 다음 **"충돌(Conflict)이 나면 어떻게 할지"** 를 정하는 방식으로 훨씬 직관적이다.

```sql
-- 🔴 [조건 2에 해당] 일단 일치하는 데이터가 없다고 가정하고 무조건 INSERT를 시도해라! (WHEN NOT MATCHED)
INSERT INTO 테이블명 (사번, 이름, 부서명) 
VALUES (101, '김맥북', 'AI팀')

-- 🟢 [조건 1에 해당] 어라? 넣으려고 보니 사번(PK)이 겹쳐서 충돌이 나네? (WHEN MATCHED)
ON CONFLICT (사번)                
-- 그럼 에러 뱉지 말고, 방금 넣으려던 그 새 값(EXCLUDED)으로 업데이트나 해줘!
DO UPDATE SET 
    부서명 = EXCLUDED.부서명, 
    연봉 = EXCLUDED.연봉;
```

>`EXCLUDED`란? INSERT 하려다 충돌 나서 튕겨 나온 새 데이터를 가리키는 키워드. Oracle의 `소스테이블 S`와 완전히 같은 역할이다

---
## 흔한실수

**"오라클 MERGE 쓸 때 `INSERT`에 `VALUES` 대신 `SELECT` 써도 되나요?"**
- **절대 안 돼!** 일반 `INSERT` 문은 대량 삽입할 때 `SELECT`를 쓸 수 있지만, `MERGE` 안의 `INSERT`는 무조건 **`VALUES`** 만 써야 해. 
- 애초에 `USING 소스테이블 S`에서 대량의 데이터를 이미 `SELECT`로 퍼왔기 때문이야!

**"`WHEN MATCHED THEN` 안에서 특정 조건일 때만 업데이트할 수도 있나요?"**
- **가능해! (SQLD 기출)** `UPDATE SET ...` 뒤에 `WHERE` 절을 또 붙일 수 있어!
- 예: `WHEN MATCHED THEN UPDATE SET T.연봉 = S.연봉 WHERE S.실적 > 90` (매칭은 됐는데, 실적까지 90 넘는 애들만 업데이트해라!)

```sql
WHEN MATCHED THEN 
    UPDATE SET T.연봉 = S.연봉 
    WHERE S.실적 > 90  -- 매칭됐어도 실적 90 초과인 경우만 업데이트
```

