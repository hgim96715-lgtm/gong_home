---
aliases:
  - read_csv
  - to_csv
  - read_parquet
  - to_parquet
  - read_excel
  - to_excel
  - read_json
  - 파일입출력
  - IO
tags:
  - Pandas
  - FileIO
related:
  - "[[Data_Formats_CSV]]"
  - "[[Data_Formats_Parquet]]"
  - "[[Text_vs_Binary]]"
  - "[[Pandas_DataStructures]]"
  - "[[Python_File_IO]]"
  - "[[Python_Database_Connect]]"
---

## 1. 개념 한 줄 요약 

**"하드디스크(Storage)에 있는 파일을 메모리(RAM) 위로 올리거나(`read`), 그 반대로 내리는(`to`) 작업."**

* **규칙:** 함수 이름이 직관적입니다.
    * 불러올 때: `{python}pd.read_{포맷명}("파일명")`
    * 저장할 때: `{python}df.to_{포맷명}("파일명")`

---
## 2. 파일 불러오기 (Read) 

가장 많이 쓰는 3대장(`csv`, `excel`, `parquet`)입니다.

### ① CSV 읽기 (`read_csv`)

가장 흔하지만 옵션 설정이 중요합니다. ([[Data_Formats_CSV]] 참고)

```python
import pandas as pd

# 1. 기본 읽기
df = pd.read_csv("data.csv")

# 2. [필수] 옵션 사용하여 읽기
df = pd.read_csv("data.csv", 
                 sep=",",           # 구분자 (탭이면 '\t')
                 header=0,          # 첫 줄이 제목임 (없으면 None)
                 encoding="utf-8")  # 한글 깨지면 "cp949" 시도!
```

> **🚨 [주의] Spark vs Pandas 헤더 옵션 차이**
> * **Spark:** `header=True` (무조건 첫 줄이 헤더)
> * **Pandas:** `header=0` (0번째 줄이 헤더).
>   * 만약 헤더가 없는 파일이라면? 👉 `header=None` (반드시 None을 써야 함. False 아님!)

**실험실: `header=1`은 언제 써요?**

[파일 모양: data.csv]

```text
---- (0번 줄) 2024년 1분기 보고서 ----  <-- 쓸모없음
이름, 나이, 부서                       <-- 진짜 헤더 (1번 줄)
김철수, 30, 영업팀
```

```python
# 0번 줄(보고서 제목)은 무시하고, 1번 줄부터 헤더로 인식함!
df = pd.read_csv("data.csv", header=1)
```


### ② 엑셀 읽기 (`read_excel`)

- **주의:** `openpyxl` 라이브러리가 설치되어 있어야 합니다.
- 시트가 여러 개면 `sheet_name`을 지정해야 합니다.

```python
# 특정 시트 읽기 (시트 이름 또는 번호)
df = pd.read_excel("data.xlsx", sheet_name="Sheet1")
```

> **💡 Tip: 시트 이름이 기억 안 날 땐?**
> `pd.ExcelFile()`을 쓰면 데이터를 다 읽지 않고 이름만 빠르게 스캔할 수 있습니다.
> ```python
> xls = pd.ExcelFile("data.xlsx")
> print(xls.sheet_names)  # ['Sheet1', 'Data', 'Result']
> ```

### ③ Parquet 읽기 (`read_parquet`) ⚡️

- **주의:** `pyarrow` 또는 `fastparquet` 라이브러리 필요.
- 속도가 CSV보다 월등히 빠릅니다. ([[Data_Formats_Parquet]] 참고)

```python
df = pd.read_parquet("data.parquet")
```

### ④ JSON 읽기 (`read_json`)

- 웹 데이터나 로그 파일 읽을 때 씁니다.
- **`lines=True`**: 로그 파일처럼 "한 줄에 JSON 하나씩" 들어있는 경우 필수 옵션입니다.

```python
# 일반적인 JSON 파일
df = pd.read_json("data.json")

# 한 줄에 하나씩 있는 로그형 JSON (NDJSON)
df = pd.read_json("log.json", lines=True)
```

### ⑤ 데이터베이스 읽기 (`read_sql`)

- **주의:** Pandas는 DB와 직접 대화할 수 없습니다. 통역사 역할을 하는 `engine` (주로 `SQLAlchemy` 라이브러리)이 반드시 필요합니다.
- **필수 라이브러리:** `SQLAlchemy` 및 DB 종류에 맞는 통신 드라이버 (예: PostgreSQL의 경우 `psycopg2`)

```python
import pandas as pd
from sqlalchemy import create_engine

# 1. Engine 생성 (DB 통역사 연결)
# 포맷: "디비종류://아이디:비밀번호@주소:포트/디비이름"
db_url = "postgresql://airflow:airflow@postgres:5432/airflow"
engine = create_engine(db_url)

# 2. 쿼리문 작성 및 데이터 읽기
query = "SELECT * FROM sales_aggregation WHERE category = '패션'"
df = pd.read_sql(query, engine)
```



---
## 파일 저장하기 (Write / To) 📤

분석한 결과를 파일로 만듭니다. 
**가장 중요한 건 `index=False` 옵션입니다.**

### ① CSV로 저장 (`to_csv`)

```python
# 🚨 [초보 실수 1위] index=False 안 쓰면?
# -> 파일 열었을 때 불필요한 행 번호(0, 1, 2...)가 첫 번째 열에 생김 (Unnamed: 0)

# ✅ [정석]
df.to_csv("result.csv", index=False, encoding="utf-8-sig")
```

**Tip:** `utf-8-sig`는 엑셀에서 열었을 때 한글이 안 깨지게 해주는 마법의 인코딩입니다.

### ② Parquet로 저장 (`to_parquet`)

```python
# 압축 방식(compression) 지정 가능 (snappy, gzip, brotli 등)
df.to_parquet("result.parquet", engine="pyarrow", compression="snappy")
```

### ③ 데이터베이스로 저장 (`to_sql`)

분석이나 집계가 끝난 데이터프레임을 DB 테이블로 바로 꽂아 넣을 때 사용합니다.

**`if_exists` 옵션:** 테이블이 이미 존재할 때 어떻게 행동할지 결정하는 핵심 옵션입니다.

- `'fail'` (기본값): 에러를 냅니다.
- `'replace'`: 기존 테이블을 싹 지우고 새로 만듭니다.
- `'append'`: 기존 데이터 밑에 새로운 데이터를 추가합니다. (실무에서 가장 많이 씀!)

```python
# df 데이터를 'sales_report' 테이블에 밀어넣기
df.to_sql(
    name="sales_report",     # DB에 저장될 테이블 이름
    con=engine,              # 연결해둔 engine 객체
    if_exists="append",      # 이미 테이블이 있으면 밑에 추가해라
    index=False              # DataFrame의 인덱스 번호는 DB에 넣지 마라
)
```


---
## [비교] Spark vs Pandas I/O 차이

| **구분**    | **Pandas (pd.read_...)**                      | **Spark (spark.read...)**                                |
| --------- | --------------------------------------------- | -------------------------------------------------------- |
| **동작 시점** | **즉시 실행 (Eager)**<br><br>함수 실행하자마자 메모리에 다 올림. | **지연 실행 (Lazy)**<br><br>"파일 위치만 기억해둠." (Action 전까지 안 읽음) |
| **메모리**   | 파일 크기가 RAM보다 크면 **터짐 (Memory Error).**        | RAM보다 커도 **나눠서 처리 가능.**                                  |
| **분산 처리** | 파일 1개를 CPU 1개가 읽음.                            | 파일 1개를 여러 조각내서 CPU 여러 개가 읽음.                             |


---
## 자주 겪는 에러 (Troubleshooting) 

### Q1. "UnicodeDecodeError: 'utf-8' codec can't decode..."

- **원인:** 한글 윈도우에서 만든 CSV(`cp949`)를 파이썬(`utf-8`)으로 읽으려 해서.
- **해결:** `pd.read_csv(..., encoding="cp949")` 추가.

### Q2. "Unnamed: 0 컬럼이 생겼어요!"

- **원인:** 저장할 때 `index=False`를 안 해서 행 번호까지 같이 저장됨.
- **해결:** 저장할 때 `df.to_csv(..., index=False)` 꼭 넣기. 이미 생겼다면 `pd.read_csv(..., index_col=0)`으로 읽기.

### Q3. "ImportError: Missing optional dependency 'openpyxl'"

- **원인:** 엑셀을 읽는 도구가 설치 안 됨.
- **해결:** `pip install openpyxl` 실행.

>"데이터 분석 결과를 팀장님한테 보낼 때는 **엑셀(`to_excel`)** 이나 **CSV(`to_csv`)** 가 좋지만, 
>네가 내일 다시 작업하려고 중간 저장할 때는 무조건 **Parquet(`to_parquet`)** 를 써! 
>속도도 빠르고, `index=False` 같은 거 신경 안 써도 되고, 데이터 타입도 그대로 보존되니까 아주 편해."


