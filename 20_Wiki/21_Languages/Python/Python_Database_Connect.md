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
  - "[[SQL_SELECT_FROM]]"
  - "[[00_Python_HomePage]]"
  - "[[SQL_DDL_Create]]"
  - "[[Python_OS_Module]]"
  - "[[Python_File_IO]]"
  - "[[Python_Virtual_Env]]"
  - "[[Streamlit_Data_Display]]"
---
## 개념 한 줄 요약

**"파이썬(Python)이 통역사(Driver)를 통해 데이터베이스(DB)와 대화하며 데이터를 넣고 빼는 기술"**

---
## 어떤 라이브러리를 써야 할까?

|         | `psycopg2`                    | `sqlalchemy`                   |
| ------- | ----------------------------- | ------------------------------ |
| 역할      | SQL 직접 실행                     | Python ↔ DB 고수준 연결             |
| 주로 쓰는 곳 | `cursor.execute()`            | `pd.read_sql()`, `df.to_sql()` |
| 특징      | 저수준, 세밀한 제어                   | 고수준, pandas 연동에 편리             |
| 실전 예시   | Kafka 로그 저장 (`save_to_db.py`) | Streamlit Dashboard DB 조회      |


---
# psycopg2

### 설치 

파이썬은 기본적으로 SQL을 모릅니다. PostgreSQL과 대화하려면 **`psycopg2`** 라는 라이브러리가 필요합니다.

```bash
# 터미널에서 실행
# binary 버전을 설치해야 컴파일 에러 없이 한방에 설치됨
pip3 install psycopg2-binary
```

---
## 기본 공식 (5단계) 

DB 작업은 항상 이 5단계를 거칩니다. (식당에 비유)

1. **`connect()`**: 식당 문 열고 들어가기 (로그인)
2. **`cursor()`**: 웨이터 부르기 (주문 받을 사람)
3. **`execute()`**: 주문하기 (SQL 전송)
4. **`commit()`**: **결제하기 (⭐ 이거 안 하면 주방에 주문 안 들어감!)**
5. **`close()`**: 문 닫고 나오기 (연결 종료)
----
## 핵심 동작 상세 해부 (The Core Actions)

DB 연결 코드는 항상 **연결(connect) -> 실행(execute) -> 확정(commit) -> 에러시 취소(rollback) -> 종료(close)**

### ① `conn = psycopg2.connect(...)` (연결)

**의미:** 식당 문을 열고 들어갑니다. (로그인)
-  **주의:** `conn`은 연결 통로일 뿐, 직접 주문을 받지는 않습니다.

### ② `cur = conn.cursor()` (웨이터 부르기,커서생성) 

**의미:** DB와 대화할 **'대리인(Cursor)'** 을 만드는 과정입니다.
실제로 주문을 받고 음식을 나르는 건 웨이터(`cur`)가 합니다. 앞으로의 모든 명령(`execute`)은 이 `cur`에게 시킵니다.

### ③ `cur.execute(sql, values)` (주문하기,실행)

**의미:** 웨이터에게 **"이 SQL(메뉴)대로 요리해줘, 재료(값)는 이거야"** 라고 전달하는 것입니다.
**구조:** `execute( SQL문장, (데이터1, 데이터2, ...) )`
- **앞쪽(SQL):** `INSERT INTO ... VALUES (%s, %s)` -> 구멍(`%s`)이 뚫린 주문서
- **뒤쪽(Values):** `(data['uuid'], data['price'])` -> 구멍을 채울 재료들(튜플 형태)

### ④ `conn.commit()` (결제/저장 확정) 성공시 

**의미:** **"지금까지 한 거 진짜로 저장해!"** 라고 도장을 찍는 것입니다.
**주의사항:** `INSERT`, `UPDATE`, `DELETE` 처럼 데이터를 바꿀 때는 **이 줄이 없으면 말짱 도루묵**입니다.
- `execute`만 하고 프로그램을 끄면? -> DB는 "아, 주문하다가 나갔네?" 하고 내용을 다 취소(Rollback)해버립니다.

### ⑤ `conn.rollback()` (주문 취소/되돌리기) 에러시 

**위치:** `except` 블록 (에러가 났을 때만 실행)
**의미:** **"방금 하던 거 없던 일로 해! (Undo)"**

### ⑥ `cur.close()` & `conn.close()` (뒷정리/퇴장) 무조건!

**위치:** `finally` 블록 (성공하든 실패하든 무조건 실행)
- **`cur.close()`**: 웨이터 퇴근시키기. (메모리 정리)
- **`conn.close()`**: **전화 끊기 (가장 중요**)

>`close()`를 안 하고 프로그램을 끄면, DB 입장에서는 **"어? 아직 전화 안 끊었네?"** 하고 자리를 계속 비워둡니다.
> 이게 쌓이면 **"Too many connections"** 에러가 뜨면서 **DB가 먹통**이 됩니다.


---
## DB 연결 설정 관리 (`DB_CONFIG`)

데이터베이스 접속 정보(Host, ID, PW 등)는 코드 곳곳에 흩어지게 두지 않고, **딕셔너리(Dictionary)** 형태로 한곳에 모아서 관리하는 것이 표준입니다.

```python
# 1. 연결 정보 딕셔너리
DB_CONFIG = {
    "host": "localhost",     # 집 주소 (내 컴퓨터)
    "database": "airflow",     # 방 이름 (데이터베이스 이름)
    "user": "airflow",       # 내 이름 (아이디)
    "password": "airflow",   # 열쇠 (비밀번호)
    "port": 5433             # 문 번호 (포트)
}
```

- **비유:** 친구 집에 놀러 가려면 **주소, 비밀번호, 호수**를 알아야 하죠? 매번 함수에 적기 귀찮으니까 `DB_CONFIG`라는 가방에 미리 다 싸놓은 겁니다.

>database를 db로바꾸니 에러가남
>이유: `psycopg2.connect()` 함수가 내부적으로 정의될 때, 데이터베이스 이름을 받기 위해 약속한 매개변수(Parameter) 이름이 **`database`** 또는 **`dbname`** 이기 때문입니다.

### 사용 가능한 '정확한 이름표' 목록

`psycopg2`와 대화할 때 쓸 수 있는 공식 이름표들은 다음과 같습니다:

| **딕셔너리 Key (이름표)**               | **설명**                   |
| -------------------------------- | ------------------------ |
| **`host`**                       | 접속 주소                    |
| **`port`**                       | 포트 번호                    |
| **`database`** (또는 **`dbname`**) | **데이터베이스 이름** (db는 안 됨!) |
| **`user`**                       | 사용자 아이디                  |
| **`password`**                   | 비밀번호                     |

---
## 쿼리 작성의 핵심 (`%s` 플레이스홀더)

SQL 쿼리에 파이썬 변수(데이터)를 넣을 때 사용하는 규칙입니다.

### 무조건 `%s`로 통일!

C언어나 다른 곳에서는 숫자는 `%d`, 문자는 `%s`로 나누지만, **`psycopg2`는 무조건 `%s`만 씁니다.**
- 숫자(`int`)가 들어오면 → 따옴표 없이 숫자로 변환 (`100`)
- 문자(`str`)가 들어오면 → 자동으로 따옴표를 붙여 변환 (`'apple'`)
- 날짜(`datetime`)가 들어오면 → DB 포맷에 맞춰 변환 (`'2024-01-01'`)

### 따옴표 절대 금지! 🚫 (가장 많이 하는 실수)

`%s` 자체가 "알아서 타입을 맞춰준다"는 뜻이므로, **개발자가 직접 따옴표를 붙이면 안 됩니다.**

- ❌ **틀린 예:** `VALUES ('%s')`
    - 이렇게 하면 숫자가 들어와도 `'5000'`(문자열)이 되어버려 에러가 납니다.
        
- ✅ **맞는 예:** `VALUES (%s)`
    - 그냥 쌩으로 `%s`만 두세요. `psycopg2`가 알아서 처리합니다.

```python
# [실전 예시]
# name="김철수", age=25 일 때

# (X) 틀린 방식: f-string이나 format 쓰지 마세요! (보안 위험 + 따옴표 귀찮음)
cur.execute(f"INSERT INTO users VALUES ('{name}', {age})")

# (O) 맞는 방식: %s 쓰고, 데이터는 튜플로 따로 넘기기
sql = "INSERT INTO users (name, age) VALUES (%s, %s)"
cur.execute(sql, (name, age))
```


---
## 딕셔너리 언패킹 (`**DB_CONFIG`)

함수에 인자를 넘길 때, 일일이 적지 않고 **딕셔너리를 '풀어서(Unpack)'** 전달하는 파이썬의 핵심 문법입니다.

### 동작 원리

함수 호출 시 변수 앞에 별표 두 개(**`**`**)를 붙이면, 딕셔너리의 **Key가 '매개변수 이름'** 이 되고 **Value가 '값'** 이 되어 전달됩니다.

```python
# [방법 A] 하드코딩 (비추천) - 수정하기 힘들고 코드가 길어짐
psycopg2.connect(host="localhost", user="airflow", password="...", ...)

# [방법 B] 언패킹 (추천) - DB_CONFIG 내용물이 자동으로 매핑됨
psycopg2.connect(**DB_CONFIG)
```

---
## 안전한 연결 패턴 (`conn = None`)

DB 연결은 네트워크 문제나 비밀번호 오류로 언제든 실패할 수 있습니다.
프로그램이 멈추지 않게 하려면 **초기화(Initialization)와 예외처리(Try-Except)** 순서를 지켜야 합니다.

### ⚠️ 왜 `conn = None`을 먼저 쓰는가?

`try` 블록 안에서 연결(`connect()`)이 실패하면 `conn` 변수가 생성조차 되지 않습니다.
이 상태에서 `finally` 블록이 `conn.close()`를 시도하면 **"conn이라는 변수가 없다"** 는 2차 에러가 발생합니다.

**✅ 표준 패턴**

```python
def run_query():
    conn = None  # 1. 변수 선언 (빈 값으로 초기화)
    
    try:
        conn = psycopg2.connect(**DB_CONFIG) # 2. 연결 시도
        # ... (쿼리 실행) ...
        
    except Exception as e:
        print(e) # 에러 로그 출력
        
    finally:
        # 3. 연결이 성공했을 때만(None이 아닐 때만) 닫기
        if conn is not None:
            conn.close()
```

---
## 보안: 환경변수 사용 (`.env`)

소스코드(GitHub 등)에 비밀번호를 직접 적는 것은 보안상 매우 위험합니다. 
**`python-dotenv`** 라이브러리를 사용해 비밀번호를 숨겨야 합니다.

### 설치

```bash
pip3 install python-dotenv
```

### `.env` 파일 생성

프로젝트 최상위 폴더에 `.env` 파일을 만들고 정보를 적습니다. 
_(주의: 이 파일은 `.gitignore`에 등록하여 커밋되지 않게 해야 합니다.)_

```TOML
DB_HOST=localhost
DB_NAME=airflow
DB_USER=airflow
DB_PASSWORD=my_secret_password
DB_PORT=5432
```

### 파이썬에서 불러오기 (`os.getenv`)

```python
import os
from dotenv import load_dotenv # import 

# 1. .env 파일 로드 (환경변수로 등록됨) .env 파일 활성화
load_dotenv()

# 2. os.getenv로 값 꺼내오기
DB_CONFIG = {
    "host": os.getenv("DB_HOST"),      # .env 파일의 DB_HOST 값
    "dbname": os.getenv("DB_NAME"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD"), # 코드를 봐도 비번을 알 수 없음
    "port": os.getenv("DB_PORT")
}
```
---
## **DB 연결 코드**

```python title='DB 연결 코드 '
import os
import psycopg2
from dotenv import load_dotenv

# 환경변수 로드
load_dotenv()

# 연결 정보 설정
DB_CONFIG = {
    "host": os.getenv("DB_HOST"),
    "dbname": os.getenv("DB_NAME"),
    "user": os.getenv("DB_USER"),
    "password": os.getenv("DB_PASSWORD"),
    "port": os.getenv("DB_PORT", 5432) # 값이 없으면 기본값 5432 사용
}

def insert_log(data):
    conn = None
    try:
        # [연결] (식당 입장)
        conn = psycopg2.connect(**DB_CONFIG)
        # 커서 생성 (웨이터 호출)
        cur = conn.cursor()
        
        # SQL 틀 준비 (%s로 구멍 뚫기)
        sql = "INSERT INTO user_logs (event_uuid, price) VALUES (%s, %s)"
        # 실행 (주문 넣기)
        # execute(SQL, (값1, 값2)) 형태로 짝을 맞춰서 던져줍니다.
        cur.execute(sql, (data['uuid'], data['price']))
        
        # [확정] 커밋 (주문 확정)
        conn.commit()
        print("✅ 저장 완료")

    except Exception as e:
        print(f"❌ DB 에러: {e}")
        # [롤백] 에러 나면 없던 일로 하기 (Undo)
        if conn:
            conn.rollback() # 작업 취소
            
    finally:
        # [종료] 무조건 문 닫고 나오기 (뒷정리)
        if conn:
            cur.close()
            conn.close()
```

---
# sqlalchemy — pandas 연동 / Streamlit Dashboard

`psycopg2`보다 한 단계 위의 고수준 라이브러리다. 
`pd.read_sql()`, `df.to_sql()` 처럼 **pandas와 함께 쓸 때** 주로 사용한다.

## 설치

```bash
pip3 install sqlalchemy psycopg2-binary
```

>`sqlalchemy`는 내부적으로 `psycopg2`를 드라이버로 쓰기 때문에 함께 설치해야 한다.

---
## 연결 방법 (`create_engine`)

```python
import os
from sqlalchemy import create_engine
from dotenv import load_dotenv

load_dotenv()

db_url = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
engine = create_engine(db_url)
```

>"Streamlit은 밖에서 실행하니까 localhost!" [[Docker_Host_vs_Internal_Network]] 참고!

### URL 구조

```python
postgresql://유저:비밀번호@호스트:포트/DB명
```

### `create_engine`이란?

`psycopg2`의 `connect()`와 다르게, **`create_engine`은 실제로 연결을 여는 게 아니다.** 
"이 주소로 연결할 준비"를 해두는 **연결 공장(Factory)** 을 만드는 것이다.

```text
db_url                →  create_engine(db_url)  →  pd.read_sql(..., engine)
"어디에 연결할지            "연결 공장 세팅"            "실제로 연결해서
 주소만 적어둠"              (아직 연결 안 함)            데이터 가져오기"
```

- `engine`을 만든다고 DB에 바로 붙는 게 아니다.
- `pd.read_sql()` 같이 실제로 데이터가 필요한 순간에 연결이 일어난다.
- 한 번 만들어두면 여러 번 재사용할 수 있다. (매번 새로 만들 필요 없음)

>**psycopg2와 비교**
>`psycopg2.connect()` → 즉시 연결, 직접 관리
>`create_engine()` → 연결 준비만, 필요할 때 자동 연결


---
## 주요 사용 패턴

```python
import pandas as pd

# DB → DataFrame (읽기)
df = pd.read_sql("SELECT * FROM report_category_sales", engine)

# DataFrame → DB (쓰기)
df.to_sql("테이블명", engine, if_exists="replace", index=False)
```

---
## 실전 사용 예시 (Streamlit Dashboard)

```python
# 페이지 로드 시 DB 데이터 한 번 읽어서 화면에 표시
try:
    df_db = pd.read_sql("SELECT * FROM report_category_sales", engine)
    st.dataframe(df_db)
except Exception as e:
    st.warning(f"DB 연결 확인 필요: {e}")
```

>**💡 네트워크 주의:** Streamlit은 로컬에서 실행되므로 `DB_HOST=localhost`를 써야 한다. 
>Spark(도커 내부)에서는 `postgres` 컨테이너명으로 접근하는 것과 다르다. → [[04_Spark_Batch]] 참고
