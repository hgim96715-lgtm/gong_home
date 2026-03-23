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
---
# Python_Database_Connect — Python + PostgreSQL 연결

## 한 줄 요약

```
psycopg2    SQL 직접 실행 (저수준 / 세밀한 제어)
sqlalchemy  pandas 연동 / 고수준 연결
```

---

---

# ① psycopg2 vs sqlalchemy

|항목|psycopg2|sqlalchemy|
|---|---|---|
|수준|저수준 (SQL 직접)|고수준 (추상화)|
|주로 쓰는 곳|cursor.execute()|pd.read_sql() / df.to_sql()|
|적합한 상황|데이터 엔지니어링 / 대용량 배치|웹 서비스 / pandas 연동|
|처리 속도|빠름 (execute_values / COPY)|대용량 배치 시 느림|
|SQL 제어|SQL 직접 튜닝 가능|추상화 → 직접 제어 어려움|
|내부 관계|독립|내부적으로 psycopg2 사용|

```
SQLAlchemy ORM:
  파이썬 클래스 → DB 테이블 매핑
  user = User(name="홍길동") → INSERT 자동
  웹 서비스 단건 처리에 적합 (Django / FastAPI)

SQLAlchemy Core (create_engine):
  SQL 은 직접 쓰되 연결 관리 자동화
  pd.read_sql() / df.to_sql() 와 함께 사용
  pandas 연동 / Streamlit 대시보드

psycopg2 직접:
  SQL 전부 직접 작성
  execute_values / COPY 로 대용량 배치 처리
  데이터 엔지니어링 파이프라인에 최적

선택 기준:
  웹 서비스 단건 CRUD    → SQLAlchemy ORM
  pandas / Streamlit    → SQLAlchemy Core (create_engine)
  대용량 배치 / ETL      → psycopg2 + execute_values
```

---

---

# ② psycopg2 — SQL 직접 실행

## 설치

```bash
pip install psycopg2-binary   # binary 버전 설치 (컴파일 에러 없음)
```

## DB_CONFIG — 연결 정보 딕셔너리

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    "host":     os.getenv("DB_HOST",     "localhost"),
    "dbname":   os.getenv("DB_NAME",     "hospital_db"),  # database 또는 dbname
    "user":     os.getenv("DB_USER",     "hospital_user"),
    "password": os.getenv("DB_PASSWORD", ""),
    "port":     os.getenv("DB_PORT",     5432),
}
```

```
⚠️ "db" 키 쓰면 에러
  psycopg2.connect() 가 인식하는 키:
    host / port / dbname / database / user / password

  DB_CONFIG = {"db": "hospital_db"}  ❌
  DB_CONFIG = {"dbname": "hospital_db"}  ✅
```

## 5단계 기본 흐름

```
① connect()  → 연결 열기
② cursor()   → 커서 생성 (실제 명령 전달자)
③ execute()  → SQL 실행
④ commit()   → 변경 확정 (없으면 말짱 도루묵)
⑤ close()    → 연결 닫기 (안 하면 Too many connections)
```

## 안전한 연결 패턴 — conn = None ⭐️

```python
import psycopg2

def insert_data(data: dict):
    conn = None   # 반드시 None 으로 초기화
    try:
        # ① 연결
        conn = psycopg2.connect(**DB_CONFIG)
        cur  = conn.cursor()

        # ③ SQL 실행
        sql = "INSERT INTO er_realtime (hpid, hvec) VALUES (%s, %s)"
        cur.execute(sql, (data["hpid"], data["hvec"]))

        # ④ 커밋 (없으면 저장 안 됨!)
        conn.commit()
        print("저장 완료")

    except Exception as e:
        print(f"DB 에러: {e}")
        if conn:
            conn.rollback()   # 에러 시 되돌리기

    finally:
        if conn:
            cur.close()
            conn.close()      # 반드시 닫기
```

```
conn = None 이 필요한 이유:
  connect() 가 실패하면 conn 변수 자체가 생성되지 않음
  finally 에서 conn.close() 시도 → "conn 이 없다" 에러 발생
  → conn = None 으로 초기화해두면
    if conn: 조건으로 안전하게 체크 가능
```

## %s 플레이스홀더 — 타입 자동 변환

```python
# psycopg2 는 타입 상관없이 전부 %s 사용
# 자동으로 int / str / datetime 에 맞게 변환해줌

sql = "INSERT INTO users (name, age) VALUES (%s, %s)"
cur.execute(sql, ("김철수", 25))
# → INSERT INTO users (name, age) VALUES ('김철수', 25)

# ❌ 따옴표 직접 붙이면 안 됨
sql = "INSERT INTO users VALUES ('%s')"   # 에러 또는 오동작
```

```
❌ f-string 으로 SQL 만들기  → SQL Injection 위험
   cur.execute(f"INSERT INTO users VALUES ('{name}')")

✅ %s 플레이스홀더
   cur.execute("INSERT INTO users VALUES (%s)", (name,))
```

## 여러 건 한번에 INSERT — execute_values

```python
from psycopg2.extras import execute_values

rows = [
    ("A1100001", "서울대병원", 15),
    ("A1100002", "세브란스", 8),
]

sql = "INSERT INTO er_realtime (hpid, hpname, hvec) VALUES %s"
execute_values(cur, sql, rows)
conn.commit()
```

```
execute_values:
  rows 리스트를 한 번의 INSERT 로 처리
  건별 execute() 반복보다 훨씬 빠름
  대량 적재 시 권장
```

## 읽기 — fetchall / fetchone

```python
cur.execute("SELECT hpid, hvec FROM er_realtime WHERE hvec > 0")

# 전체 결과
rows = cur.fetchall()        # [(hpid, hvec), ...]
for row in rows:
    print(row[0], row[1])

# 한 건만
row = cur.fetchone()         # (hpid, hvec)

# 딕셔너리 형태로 받기
from psycopg2.extras import RealDictCursor
cur = conn.cursor(cursor_factory=RealDictCursor)
cur.execute("SELECT * FROM er_realtime LIMIT 5")
rows = cur.fetchall()        # [{"hpid": "...", "hvec": 15}, ...]
```

---

---

# ③ sqlalchemy — pandas 연동

## 설치

```bash
pip install sqlalchemy psycopg2-binary
# sqlalchemy 는 내부적으로 psycopg2 를 드라이버로 사용
```

## create_engine — 연결 공장

```python
from sqlalchemy import create_engine

db_url = (
    f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
    f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
)
engine = create_engine(db_url)
```

```
URL 구조:
  postgresql://유저:비밀번호@호스트:포트/DB명

create_engine() vs psycopg2.connect():
  psycopg2.connect()  → 즉시 연결 / 직접 관리
  create_engine()     → 연결 준비만 (공장 설정)
                        실제 데이터 필요할 때 자동 연결
                        한 번 만들어두면 재사용 가능
```

## pandas 연동

```python
import pandas as pd

# DB → DataFrame 읽기
df = pd.read_sql("SELECT * FROM er_realtime WHERE hvec > 0", engine)

# DataFrame → DB 쓰기
df.to_sql(
    "er_realtime_backup",
    engine,
    if_exists="replace",   # replace / append / fail
    index=False
)
```

|if_exists|동작|
|---|---|
|`replace`|테이블 삭제 후 새로 생성|
|`append`|기존 데이터에 추가|
|`fail`|이미 있으면 에러|

## Streamlit 에서 활용

```python
# Streamlit 은 로컬에서 실행 → DB_HOST=localhost
try:
    df = pd.read_sql("SELECT * FROM er_realtime LIMIT 100", engine)
    st.dataframe(df)
except Exception as e:
    st.warning(f"DB 연결 확인 필요: {e}")
```

```
⚠️ 호스트 주의:
  Streamlit (로컬 실행)  → DB_HOST=localhost
  Spark (컨테이너 안)    → DB_HOST=postgres (서비스명)
  → [[Docker_Host_vs_Internal_Network]] 참고
```

---

---

# ④ .env — 비밀번호 안전하게 관리

```bash
pip install python-dotenv
```

```bash
# .env 파일 (프로젝트 루트에 생성)
DB_HOST=localhost
DB_NAME=hospital_db
DB_USER=hospital_user
DB_PASSWORD=my_secret_password
DB_PORT=5432
```

```bash
# .gitignore 에 반드시 추가
.env
```

```python
import os
from dotenv import load_dotenv

load_dotenv()   # .env 파일 로드

host = os.getenv("DB_HOST")
pw   = os.getenv("DB_PASSWORD")
port = os.getenv("DB_PORT", 5432)  # 없으면 기본값 5432
```

---

---

# 한눈에 정리

```
psycopg2:
  pip install psycopg2-binary
  connect(**DB_CONFIG) → cursor() → execute(sql, (값,)) → commit() → close()
  conn = None 초기화 → finally if conn: close()
  대량 INSERT → execute_values

sqlalchemy:
  pip install sqlalchemy psycopg2-binary
  create_engine("postgresql://유저:pw@호스트:포트/DB명")
  pd.read_sql(sql, engine)
  df.to_sql("테이블", engine, if_exists="append")

.env:
  비밀번호 코드에 직접 ❌
  os.getenv() 로 읽기 ✅
  .gitignore 에 .env 추가 ✅
```