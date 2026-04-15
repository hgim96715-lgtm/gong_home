---
aliases:
  - DB연결
  - psycopg2
  - 파이썬-SQL
  - 데이터베이스 연동
  - sqlalchemy
  - create_engine
tags:
  - Python
  - SQL
related:
  - "[[00_Python_HomePage]]"
  - "[[PostgreSQL_Setup]]"
  - "[[Python_Functions]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[Airflow_Hooks]]"
  - "[[Linux_OpenSSL]]"
  - "[[SQL_SubQuery]]"
---
# Python_Database_Connect — Python + PostgreSQL 연결

---

## 전체 흐름 한눈에

```
① psycopg2.connect()  → DB 연결 (conn 객체 생성)
         │
         ▼
② conn.cursor()       → 커서 생성 (cur 객체 생성)
         │
         ▼
③ cur.execute(SQL)    → SQL 실행
         │
         ├─ 읽기 (SELECT) → cur.fetchone() / cur.fetchall()
         └─ 쓰기 (INSERT/UPDATE/DELETE) → cur.rowcount → conn.commit()
         │
         ▼
④ conn.close()        → 연결 닫기
```

---

## conn, cur 이 뭔가

```python
import psycopg2

conn = psycopg2.connect(**DB_CONFIG)
cur  = conn.cursor()
```

**`conn`** — DB와 열린 연결 자체. 전화를 걸어서 연결된 상태.

**`cur`** — 연결 위에서 실제로 SQL을 실행하는 객체. 전화기를 잡고 말하는 입.

```
conn = 전화 연결
cur  = 전화기 (실제로 말하는 도구)

연결(conn) 하나에 커서(cur) 여러 개 만들 수 있음
하지만 보통 커서 하나로 순서대로 사용
```

---

## cur.execute() — SQL 실행의 전부

### 기본 문법

```python
cur.execute(sql)              # 파라미터 없을 때
cur.execute(sql, params)      # 파라미터 있을 때
```

**`sql`** — SQL 문자열. 그대로 DB에 전달된다. **`params`** — SQL 안의 `%s` 자리에 들어갈 값. 튜플로 감싸야 한다.

### SQL을 문자열로 넣는 방법

```python
# 방법 1: 직접 문자열 (짧을 때)
cur.execute("SELECT COUNT(*) FROM raw_exhibitions WHERE is_active = TRUE")

# 방법 2: 변수에 담아서 (길 때)
sql = "SELECT COUNT(*) FROM raw_exhibitions WHERE is_active = TRUE"
cur.execute(sql)

# 방법 3: 삼중 따옴표로 여러 줄 (복잡한 쿼리)
sql = """
    SELECT location, COUNT(*) AS cnt
    FROM raw_exhibitions
    WHERE is_active = TRUE
    GROUP BY location
    ORDER BY cnt DESC
"""
cur.execute(sql)
```

> SQL 끝의 `;` 세미콜론은 있어도 되고 없어도 된다.

---

## execute() 후 결과 가져오기 — fetch

`execute()`는 SQL을 실행할 뿐, 결과를 돌려주지 않는다. 결과는 `fetch` 메서드로 따로 가져와야 한다.

```python
cur.execute("SELECT COUNT(*) FROM raw_exhibitions")
# ← 여기까지는 아무 값도 없음

row = cur.fetchone()   # ← 이때 결과를 가져옴
```

### fetchone() — 1행만

```python
cur.execute("SELECT COUNT(*) FROM raw_exhibitions WHERE is_active = TRUE")
row = cur.fetchone()
# row = (47,)   ← 결과가 1행, 1컬럼 → 값 하나짜리 튜플

count = row[0]   # 47  ← [0]으로 첫 번째 값 꺼냄
```

```
fetchone() 은 항상 튜플을 반환한다

SELECT COUNT(*)          → (47,)
SELECT AVG(a), AVG(b)    → (18347.0, 13160.0)
SELECT name, age         → ('홍길동', 25)

[0] = 첫 번째 컬럼
[1] = 두 번째 컬럼
```

```python
# 실제 패턴 — 두 줄을 한 줄로 줄이기
stats["active"] = cur.fetchone()[0]
# cur.fetchone()   → (47,)
# [0]              → 47

# 여러 컬럼일 때
cur.execute("""
    SELECT
        ROUND(AVG(price_adult)::NUMERIC, 0),
        ROUND(AVG(price_youth)::NUMERIC, 0)
    FROM raw_exhibitions
    WHERE is_active = TRUE AND price_adult > 0
""")
row = cur.fetchone()           # (18347.0, 13160.0)
avg_adult = row[0]             # 18347.0
avg_youth = row[1]             # 13160.0

# None 방어 처리 (집계 대상이 없으면 None 반환)
avg_adult = int(row[0]) if row and row[0] else 0
```

### fetchall() — 전체 행

```python
cur.execute("""
    SELECT location, COUNT(*) AS cnt
    FROM raw_exhibitions
    WHERE is_active = TRUE
    GROUP BY location
    ORDER BY cnt DESC
    LIMIT 5
""")
rows = cur.fetchall()
# rows = [('서울', 30), ('부산', 5), ('대구', 3), ...]
#           ↑ 튜플들의 리스트

for loc, cnt in rows:    # 튜플 언패킹
    print(f"{loc}: {cnt}개")
```

```
fetchone()  → 튜플 1개    (47,)
fetchall()  → 튜플 리스트  [('서울', 30), ('부산', 5)]
fetchmany(n) → 튜플 n개    (잘 안 씀)
```

---

## execute() 에 파라미터 넣기 — %s

SQL 안에 값을 직접 쓰지 않고 `%s` 자리를 만들어두고 따로 넘긴다.

```python
# ❌ f-string 으로 SQL 만들기 → SQL Injection 위험
name = "'; DROP TABLE users; --"
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ %s 플레이스홀더 → psycopg2가 안전하게 처리
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
```

### %s 문법 핵심

```python
cur.execute(sql, params)
#                ↑
#    반드시 튜플 또는 리스트로 감싸야 함
```

**값 1개 — 쉼표 필수**

```python
# ("001") 은 괄호만 있는 것 → 그냥 "001" 문자열
# ("001",) 은 원소 1개짜리 튜플

cur.execute("SELECT * FROM 테이블 WHERE id = %s", ("001",))  # ✅
cur.execute("SELECT * FROM 테이블 WHERE id = %s", ("001"))   # ❌ 문자열로 해석
```

**값 여러 개 — 순서대로 매핑**

```python
cur.execute(
    "INSERT INTO 테이블 (a, b, c) VALUES (%s, %s, %s)",
    ("001", "전시A", 15000)
    #  ↑       ↑       ↑
    # 첫%s   둘째%s  셋째%s  → 순서대로 매핑
)
```

**리스트 통째로 — ALL / ANY 배열용**

```python
active_ids = ["001", "002", "003"]

cur.execute(
    "UPDATE 테이블 SET is_active = FALSE WHERE id != ALL(%s)",
    (active_ids,)
#    ↑         ↑
#  리스트      ,  ← "리스트 하나를 파라미터 1개로 넘긴다"
)
# psycopg2가 리스트를 PostgreSQL 배열로 자동 변환
# → ARRAY['001', '002', '003']
```

```python
# 헷갈리는 케이스 4가지 비교
ids = ["001", "002", "003"]

cur.execute("WHERE id != ALL(%s)", (ids,))          # ✅ 배열로 변환
cur.execute("WHERE id NOT IN %s", (tuple(ids),))    # ✅ IN 절용 튜플
cur.execute("WHERE id != ALL(%s)", ids)             # ❌ 파라미터 3개로 해석됨
cur.execute("WHERE id = %s", (ids[0],))             # ✅ 첫 원소 하나만
```

| 상황         | 전달 방법            | SQL 변환 결과                    |
| ---------- | ---------------- | ---------------------------- |
| 단일 값       | `("001",)`       | `{text}= '001'`              |
| IN 절       | `(tuple(ids),)`  | `IN ('a','b','c')`           |
| ALL/ANY 배열 | `(ids,)`         | `!= ALL(ARRAY['a','b','c'])` |
| 여러 값 순서대로  | `("a", 1, True)` | 각 `%s`에 순서대로                 |

---

## cur.rowcount — 영향받은 행 수

쓰기 SQL(INSERT/UPDATE/DELETE) 실행 후, 실제로 영향받은 행 수를 반환.

```python
cur.execute("""
    UPDATE raw_exhibitions
    SET is_active = FALSE
    WHERE exhibition_id != ALL(%s)
      AND is_active = TRUE
""", (active_ids,))

updated = cur.rowcount   # ← execute() 바로 다음에 읽기
conn.commit()

if updated > 0:
    print(f"  {updated}개 비활성화")
```

```
cur.rowcount 반환값:
  INSERT → 삽입된 행 수
  UPDATE → 변경된 행 수
  DELETE → 삭제된 행 수
  SELECT → 조회된 행 수 (일부 드라이버는 -1 반환)

⚠️ 주의:
  commit() 전에 읽어야 함
  -1이 나오면 드라이버가 행 수를 파악 못한 것 (SELECT에서 종종 발생)
```

---

## 쓰기 SQL — commit() 필수

SELECT는 읽기만 하므로 commit 불필요. INSERT / UPDATE / DELETE는 **반드시 commit()** 해야 저장된다.

```python
# commit 없으면 아무것도 안 저장됨
cur.execute("UPDATE raw_exhibitions SET is_active = FALSE WHERE ...")
# ← 여기까지는 임시 상태 (아직 DB에 반영 안 됨)

conn.commit()   # ← 이때 비로소 DB에 영구 저장

# 에러 시 되돌리기
conn.rollback()  # commit 전 변경사항 취소
```

---

## 안전한 전체 패턴 — conn = None ⭐️

```python
conn = None   # 반드시 None 으로 초기화
try:
    conn = psycopg2.connect(**DB_CONFIG)
    with conn.cursor() as cur:    # with 블록: cur.close() 자동
        cur.execute(sql, params)
        result = cur.fetchone()   # 읽기일 때
        # 또는
        updated = cur.rowcount    # 쓰기일 때
    conn.commit()                 # 쓰기일 때만

except Exception as e:
    print(f"DB 에러: {e}")
    if conn:
        conn.rollback()           # 에러 시 되돌리기
    raise

finally:
    if conn:
        conn.close()              # 반드시 닫기
```

```
conn = None 이 필요한 이유:
  psycopg2.connect() 실패 시 → conn 변수 자체가 생성 안 됨
  finally에서 conn.close() 시도 → NameError 발생
  → None으로 초기화해두면 if conn: 으로 안전하게 체크 가능
```

```
with conn.cursor() as cur: 와 conn.close() 의 차이:
  with conn.cursor() → cur.close() 자동 (커서만)
  conn.close()       → 직접 호출 필요 (커넥션 자체)

  with conn:         → 트랜잭션 관리 (commit/rollback)
                        커넥션 닫기는 아님
```

---

## execute_values — 대량 INSERT ⭐️

`execute()` 반복보다 훨씬 빠름. 1000건도 SQL 1번으로 처리.

```python
from psycopg2.extras import execute_values

values = [
    ("26003180", "인상주의를 넘어", "서울"),
    ("26002594", "페르난도 보테로展", "서울"),
    ("26003427", "맥스 시덴토프 개인전", "부산"),
]

sql = """
INSERT INTO raw_exhibitions (exhibition_id, title, location)
VALUES %s
ON CONFLICT (exhibition_id) DO UPDATE SET
    title    = EXCLUDED.title,
    location = EXCLUDED.location
"""

conn = psycopg2.connect(**DB_CONFIG)
with conn.cursor() as cur:
    execute_values(cur, sql, values)
conn.commit()
conn.close()
```

```
execute_values(cur, sql, values)
                 ↑    ↑    ↑
                커서   SQL  행 리스트 (각 행은 튜플)

VALUES %s → VALUES (%s,%s,%s), (%s,%s,%s), (%s,%s,%s) 로 자동 확장
→ 한 번의 쿼리로 전체 삽입

EXCLUDED: ON CONFLICT 블록 안에서 "방금 삽입 시도한 값"
  EXCLUDED.title = 충돌이 난 행의 새 title 값
  → 충돌 시 이 값으로 덮어씀
```

```python
# execute() 반복 vs execute_values 속도 비교
for row in values:                          # ❌ 1000건 = 1000번 왕복
    cur.execute("INSERT INTO t VALUES (%s, %s)", row)

execute_values(cur, "INSERT INTO t VALUES %s", values)  # ✅ 1번 왕복
```

---

## 임포트 정리

```python
import psycopg2                              # 기본 연결
from psycopg2.extras import execute_values   # 대량 INSERT
from psycopg2.extras import RealDictCursor   # 결과를 dict로
```

**`psycopg2.extras`** — psycopg2 확장 모듈. `execute_values`, `RealDictCursor` 등 편의 기능이 여기 있다.

---

## RealDictCursor — 결과를 dict로

```python
from psycopg2.extras import RealDictCursor

with conn.cursor(cursor_factory=RealDictCursor) as cur:
    cur.execute("SELECT exhibition_id, title FROM raw_exhibitions LIMIT 3")
    rows = cur.fetchall()

# 기본 cursor:     [('001', '전시A'), ('002', '전시B')]
# RealDictCursor: [{'exhibition_id': '001', 'title': '전시A'}, ...]

for row in rows:
    print(row['title'])   # 컬럼명으로 접근
```

---

## 타입 힌트

```python
def get_connection(self) -> psycopg2.extensions.connection:
    return psycopg2.connect(**self.db_config)

def upsert(self, exhibitions: List[Dict[str, Any]]) -> int:
```

```
psycopg2.extensions.connection
  → psycopg2.connect()가 반환하는 객체의 타입 이름
  → IDE 자동완성 목적. 실행에 영향 없음

List[Dict[str, Any]]
  → 딕셔너리들의 리스트
  → Dict[str, Any]: 키=str, 값=아무 타입

Python 3.10+: list[dict[str, Any]] (소문자 가능)
```

---

## .get() vs [] — dict에서 값 꺼낼 때

```python
ex = {"exhibition_id": "001", "title": "전시A"}

ex["title"]              # "전시A"
ex["price_adult"]        # KeyError ← 없는 키

ex.get("title")          # "전시A"
ex.get("price_adult")    # None ← 에러 없음
ex.get("is_active", True) # True ← 없으면 기본값
```

```python
# INSERT 값 만들 때 .get() 쓰는 이유:
# 모든 전시가 모든 필드를 가지지 않음 → 없으면 None → DB에 NULL
values = [
    (
        ex["exhibition_id"],       # PK: 반드시 있어야 해서 []
        ex.get("title"),           # 없으면 None → NULL
        ex.get("price_adult"),     # 없으면 None → NULL
        ex.get("is_active", True), # 없으면 기본값 True
    )
    for ex in exhibitions
]
```

---

## DB_CONFIG — 연결 정보

```python
import os
from dotenv import load_dotenv

load_dotenv()   # .env 파일 로드

DB_CONFIG = {
    "host":     os.getenv("POSTGRES_HOST", "localhost"),
    "port":     int(os.getenv("POSTGRES_PORT", "5432")),
    "dbname":   os.getenv("POSTGRES_DB"),      # 키 이름: dbname
    "user":     os.getenv("POSTGRES_USER"),
    "password": os.getenv("POSTGRES_PASSWORD"),
}
```

```
⚠️ psycopg2 인식하는 키:
  host / port / dbname / database / user / password

  "db"     → KeyError ❌
  "dbname" → 정상 ✅
```

---

## sqlalchemy — pandas 연동

psycopg2 직접 사용 없이 pandas와 연결할 때.

```python
from sqlalchemy import create_engine

engine = create_engine(
    f"postgresql://{os.getenv('POSTGRES_USER')}:{os.getenv('POSTGRES_PASSWORD')}"
    f"@{os.getenv('POSTGRES_HOST')}:{os.getenv('POSTGRES_PORT')}/{os.getenv('POSTGRES_DB')}"
)

df = pd.read_sql("SELECT * FROM raw_exhibitions", engine)   # DB → DataFrame
df.to_sql("backup", engine, if_exists="append", index=False) # DataFrame → DB
```

|`if_exists`|동작|
|---|---|
|`replace`|테이블 삭제 후 새로 생성|
|`append`|기존 데이터에 추가|
|`fail`|이미 있으면 에러|

---
