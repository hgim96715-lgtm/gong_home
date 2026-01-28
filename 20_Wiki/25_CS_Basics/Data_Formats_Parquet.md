---
aliases:
  - Parquet
  - 파케이
  - ColumnarStorage
  - 열지향저장
  - Snappy
tags:
  - FileFormat
  - CS
related:
  - "[[Pandas_Read_Write]]"
  - "[[Spark_File_IO_Basic]]"
  - "[[Data_Formats_CSV]]"
  - "[[00_CS_HomePage]]"
---
## 개념 한 줄 요약 

**"빅데이터 처리를 위해 태어난, 압축률 쩔고 속도 엄청 빠른 '바이너리(Binary)' 포맷."**

* **읽기 전용:** 메모장으로 열면 외계어(`@^...`)가 나옵니다. (사람용 X, 기계용 O)
* **컬럼 기반 (Columnar):** 데이터를 **세로(열) 방향**으로 묶어서 저장합니다. (핵심 )
* **스파크의 단짝:** Apache Spark의 기본(Default) 저장 포맷입니다.

---
## 왜 CSV보다 10배 빠른가요? (Row vs Columnar) 

이 원리를 이해하는 게 **데이터 엔지니어링의 핵심**입니다.

### ① 행 기반 (Row-Oriented) - CSV

데이터를 **한 줄씩** 가로로 쌓습니다.

> "1번 사람의 이름, 나이, 주소... 2번 사람의 이름, 나이, 주소..."

* **단점:** "나이 평균 구해줘"라고 하면, 필요 없는 이름과 주소까지 다 읽어야 합니다.

### ② 열 기반 (Columnar-Oriented) - Parquet

데이터를 **항목별**로 세로로 묶어서 쌓습니다.

> "이름들 쫙 모음... 나이들 쫙 모음... 주소들 쫙 모음..."

* **장점:** "나이 평균 구해줘"라고 하면, **'나이' 뭉치만 쏙 꺼내서 계산**합니다. (나머지 90% 데이터는 쳐다도 안 봄 -> **속도 폭발 🚀**)

---
## Parquet의 3가지 슈퍼 파워 

### ① 압축률 (Size) 

CSV 대비 용량이 **1/3 ~ 1/10**로 줄어듭니다.
* **이유:** 비슷한 데이터끼리 모여 있으니까요!
* 예: `Seoul, Seoul, Seoul...` -> "Seoul이 100개 있음"으로 저장해버림. (RLE 압축)

### ② 스키마 보존 (Schema) 

CSV의 고질병인 "숫자가 문자로 바뀌는 문제"가 없습니다.
* 파일 자체에 **"이 컬럼은 정수(Integer)야!"** 라는 정보(메타데이터)가 박혀 있습니다.
* `inferSchema=True` 같은 거 안 해도 타입을 정확히 압니다.

### ③ 컬럼 가지치기 (Column Pruning) 

SQL에서 `SELECT Age FROM Table`을 날리면,
Parquet는 파일 전체를 안 읽고 **Age 컬럼만 딱 읽어서** 메모리에 올립니다. (I/O 비용 급감)

---
## Parquet vs CSV 비교표 

| 구분 | **CSV (.csv)** | **Parquet (.parquet)** |
| :--- | :--- | :--- |
| **가독성** | **사람이 읽기 좋음** (메모장 가능) | **기계가 읽기 좋음** (전용 툴 필요) |
| **저장 방식** | 행 (Row) 기반 | **열 (Column) 기반** |
| **용량** | 큼 (Text 그대로 저장) | **작음** (고효율 압축) |
| **처리 속도** | 느림 (전체 스캔) | **빠름** (필요한 컬럼만 스캔) |
| **스키마** | 없음 (추론 필요) | **내장됨** (완벽 보존) |
| **용도** | **최종 리포트, 데이터 교환** | **중간 데이터 저장, 분석용 DB** |

---
## 사용법 (Pandas & Spark) 

### ① Pandas

`pyarrow` 또는 `fastparquet` 라이브러리가 설치되어 있어야 합니다.

```python
# 저장하기 (압축 방식: snappy, gzip 등)
df.to_parquet("data.parquet", engine="pyarrow", compression="snappy")

# 불러오기
df = pd.read_parquet("data.parquet")
```

### ② Spark (Native Support)

스파크는 Parquet를 가장 좋아합니다. 별도 설정 없이 그냥 씁니다.

```python
# 저장하기
df.write.parquet("data.parquet")

# 불러오기 (스키마 자동 인식)
df = spark.read.parquet("data.parquet")
```

>"현업 데이터 파이프라인의 국룰을 알려줄게.
>1. 외부에서 데이터가 들어온다 (**CSV, JSON**)
>2. 일단 읽어서 **Parquet**로 변환해서 저장한다. (Raw Data Layer)
>3. 지지고 볶는 분석 작업은 전부 **Parquet** 파일을 읽어서 수행한다.
>4. 마지막에 높으신 분들한테 보낼 결과물만 **CSV/Excel**로 저장한다.

**'분석용 데이터는 무조건 Parquet'** 라고 외우면 돼. 디스크 값도 아끼고 퇴근 시간도 빨라질 거야!"

---
## CSV-> Parquet 변환하는 법

CSV로 읽어서(Read) → Parquet로 저장(Write)

### PySpark로 변환하기 ️ (추천)

빅데이터라면 스파크가 압도적으로 빠릅니다.

```python
### [실전] CSV -> Parquet 변환 파이프라인

# 1. 원본 읽기 (CSV)
raw_df = spark.read.csv("/raw/data.csv", header=True, inferSchema=True)

# 2. (선택) 데이터 다듬기 - 컬럼 이름 변경 등
clean_df = raw_df.withColumnRenamed("Age", "User_Age")

# 3. 최적화 포맷으로 저장 (Parquet)
clean_df.write.mode("overwrite").parquet("/processed/data.parquet")
```

**💡 꿀팁: `.mode("overwrite")`** 실습할 때 코드를 여러 번 돌리게 되는데,
이 옵션이 없으면 _"이미 파일이 있는데요?"_ 하고 에러가 납니다. 
"있으면 덮어써!"라고 명령하는 옵션입니다.

### Pandas로 변환하기 

데이터가 메모리(RAM)에 들어갈 정도라면 판다스도 훌륭합니다. 
단, 엔진(라이브러리)이 필요해서 **설치**가 필요할 수 있습니다.

① 설치 (터미널)

```bash
pip install pyarrow  # 또는 fastparquet
```

② 코드

```python
import pandas as pd

# 1. CSV 읽기
df = pd.read_csv("data.csv")

# 2. Parquet로 저장하기
# engine='pyarrow'를 명시해주는 게 좋습니다.
df.to_parquet("data.parquet", engine="pyarrow", compression="snappy")
```

---

### 현업의 "국룰" 폴더 구조 

그냥 변환만 하는 게 아니라, **저장하는 위치**를 분리하는 게 엔지니어링의 핵심입니다.

- **📁 /Raw_Zone (랜딩 존):** 외부에서 막 들어온 더러운 CSV들이 쌓이는 곳. (여기는 건드리지 않음)
- **📁 /Trusted_Zone (가공 존):** 위 코드를 돌려서 **Parquet로 변환된 깨끗한 파일**만 모아두는 곳.

**"앞으로 분석할 땐 무조건 `Trusted_Zone`에 있는 Parquet만 읽어라!"** 라고 팀원들에게 공지하면, 속도도 빠르고 타입 에러도 없는 평화로운 세상이 옵니다. 🕊️

