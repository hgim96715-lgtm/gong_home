---
aliases:
  - Iceberg 설치
  - Iceberg Docker
  - 스파크 아이스버그 연동
  - 실습 환경 구축
tags:
  - Spark
  - Iceberg
  - Docker
  - Setup
related:
  - "[[Apache_Iceberg_Intro]]"
  - "[[Spark_Kafka_Docker_Setup]]"
  - "[[00_Apache_Spark_HomePage]]"
---

## 개념 한줄 요약

Apache Iceberg는 별도의 서버를 설치하는 것이 아니라, **Spark 엔진에 "라이브러리(.jar)"와 "설정(Config)"을 주입**하여 기능을 활성화하는 방식입니다.
Docker 환경에서는 **Jupyter 컨테이너의 환경 변수**를 수정하여, 노트북 실행 시 자동으로 Iceberg가 로드되도록 설정합니다.

---
## Why: 왜 환경 변수(`PYSPARK_SUBMIT_ARGS`) 방식을 쓰는가?

* **편의성:** 노트북 코드마다 매번 `.config("spark.jars.packages", ...)`를 칠 필요가 없습니다.
* **표준화:** 팀원 모두가 `docker-compose` 파일 하나만 공유하면 동일한 Iceberg 환경을 가집니다.
* **유지보수:** Iceberg 버전을 올릴 때 Docker 설정 파일 한 줄만 고치면 됩니다.

##  Practical Context: 설정 가이드 (Step-by-Step)

### Step 1: `docker-compose.yaml` 수정

`jupyter` 서비스 부분에 `PYSPARK_SUBMIT_ARGS` 환경 변수를 추가합니다. (나머지 Kafka, Cassandra 등은 유지)

> **📚 공식 문서 참고:**
> 이 설정은 [Iceberg Quickstart: Adding a Catalog](https://iceberg.apache.org/spark-quickstart/#adding-a-catalog)의 내용을 Docker 환경변수 포맷으로 변환한 것입니다.

```yaml
  # 3. Jupyter Lab (Driver & Client)
  jupyter:
    # ... (기존 설정 유지) ...
    environment:
      - SPARK_MASTER=spark://spark-master:7077
      # 👇 [핵심] Iceberg 라이브러리와 카탈로그 설정 주입 (한 줄로 작성)
      - PYSPARK_SUBMIT_ARGS=--packages org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.4.3 --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions --conf spark.sql.catalog.my_iceberg=org.apache.iceberg.spark.SparkCatalog --conf spark.sql.catalog.my_iceberg.type=hadoop --conf spark.sql.catalog.my_iceberg.warehouse=/workspace/data/iceberg_warehouse pyspark-shell
```

###  설정값 해부 (Configuration Anatomy)

|**설정 옵션 (--conf)**|**설명**|**의미**|
|---|---|---|
|`spark.sql.extensions`|**Iceberg 확장 기능 활성화**|Spark SQL 문법을 확장하여 `UPDATE`, `DELETE`, `MERGE` 및 시간 여행 쿼리를 가능하게 함.|
|`spark.sql.catalog.my_iceberg`|**카탈로그 이름 지정**|SQL에서 `my_iceberg.db.table` 처럼 부를 이름을 정의함. (내 맘대로 변경 가능)|
|`...catalog.my_iceberg.type`|**저장소 타입 (hadoop)**|별도의 DB 없이 **로컬 파일 시스템**을 메타데이터 저장소로 씀. (운영 환경에선 `hive`나 `rest` 사용)|
|`...catalog.my_iceberg.warehouse`|**데이터 저장 경로**|실제 Parquet 파일이 저장될 위치. Docker Volume(`- ./data:/workspace/data`)과 연결된 경로여야 파일이 사라지지 않음.|

### Step 2: 변경 사항 적용 (재시작)

설정 파일을 고쳤으니 컨테이너를 새로 띄워야 합니다.

```bash
# Jupyter 컨테이너만 재생성 (데이터는 보존됨)
docker-compose up -d jupyter
```

---
## 코드 실습 

###  Spark Session 설정

```python
from pyspark.sql import SparkSession

spark=(
	SparkSession
	.builder
	.appName("apache-iceberg")
	.master("spark://spark-master:7077")
	.getOrCreate()
)
spark
```

### 2. 데이터베이스 및 테이블 생성

```python
# 1. DB 생성 (my_iceberg라는 카탈로그 안에 playground 생성)
# 1. DB 생성 (카탈로그명.DB명) 
# Docker 설정(--conf spark.sql.catalog.my_iceberg=...)에 맞춰서
spark.sql("CREATE DATABASE IF NOT EXISTS my_iceberg.playground")

# 2. 테이블 생성
# 일반적인 SQL과 같지만 뒤에 'USING iceberg'가 붙습니다.
# 이제 이 테이블은 Iceberg가 메타데이터를 관리합니다.
spark.sql("""
CREATE TABLE IF NOT EXISTS my_iceberg.playground.sample_table(
    id BIGINT,
    data STRING
)
USING iceberg
""")
```

>**Q. 왜 하필 `my_iceberg`라고 쓰나요?**
>**A. 우리가 Docker 환경 설정에서 그렇게 이름을 지어줬기 때문입니다!**
>파이썬 코드(Session)가 아니라, **Docker 컨테이너를 실행할 때** `--conf spark.sql.catalog.my_iceberg=...` 옵션으로 "이 창고의 이름은 my_iceberg다"라고 선언했었죠?
>그래서 SQL을 짤 때도 **"그 my_iceberg 창고에서 찾아와!"** 라고 정확히 주소를 불러주는 것입니다.


### 3. 데이터 입력 (1차)

- 기본 문법: **`{python}df.writeTo("카탈로그.DB.테이블명").명령어()`**

| 명령어                          | 의미          | 비유 (일상 언어)                                                      |  위험도  |
| :--------------------------- | :---------- | :-------------------------------------------------------------- | :---: |
| **`.append()`**              | **덧붙이기**    | "기존 내용은 그대로 두고, 맨 뒤에 새 줄을 추가해줘."                                | 🟢 안전 |
| **`.overwritePartitions()`** | **부분 교체**   | "2024년 1월 데이터만 수정됐으니까, **1월 페이지만** 찢고 다시 끼워줘." (나머지 달은 건드리지 않음) | 🟡 주의 |
| **`.create()`**              | **신규 생성**   | "새 공책을 사서 적어줘." (이미 공책이 있으면 에러 남)                               | 🟢 안전 |
| **`.createOrReplace()`**     | **완전 덮어쓰기** | "기존 공책은 찢어서 버리고, **완전히 새 공책**으로 다시 써줘." (기존 데이터 삭제됨)            | 🔴 위험 |

```python
data = [
    (1, "Hello World"),
    (2, "Apache Iceberg"),
    (3, "PySpark Demo")
]
df = spark.createDataFrame(data, ["id", "data"])

# 데이터를 Iceberg 테이블에 덧붙입니다 (Append)
# 해석: "my_iceberg... 테이블에게(writeTo)" + "데이터를 덧붙여줘(append)"
# `df.writeTo("이름").append()`
df.writeTo("my_iceberg.playground.sample_table").append()

# 잘 들어갔는지 확인
spark.sql("SELECT * FROM my_iceberg.playground.sample_table").show()
```

### 4. 스키마 변경 (Schema Evolution)

**Iceberg의 강력함 1:** 테이블을 지우지 않고 컬럼만 쏙 추가합니다.

```python
# 매번 'my_iceberg' 치기 귀찮으니, 기본 카탈로그로 설정합니다.
spark.sql("USE my_iceberg")

# 'extra_info' 컬럼 추가! (기존 데이터 파일은 건드리지 않아 매우 빠름)
spark.sql("ALTER TABLE playground.sample_table ADD COLUMN extra_info STRING")
```

### 5. 데이터 입력 (2차 - 컬럼 추가 후)

변경된 구조(3개 컬럼)에 맞춰 새 데이터를 넣습니다.

```python
data_new = [
    (4, "Another record", "This is extra info"),
    (5, "Yet another record", "Additional details")
]
df_new = spark.createDataFrame(data_new, ["id", "data", "extra_info"])
df_new.writeTo("playground.sample_table").append()

# 결과 확인: 옛날 데이터(id 1~3)는 extra_info가 NULL로 보입니다.
spark.sql("SELECT * FROM playground.sample_table").show()
```

### 6. 스냅샷 확인 (Vertical 모드)

데이터 변경 이력을 확인합니다. 
**가로로 긴 데이터는 `vertical=True`로 보는 것이 정신 건강에 좋습니다.**

```python
# .snapshots: 이 테이블의 '역사책' 같은 메타데이터
# truncate=False: 글자 자르지 마
# vertical=True: 영수증처럼 세로로 길게 보여줘 (터미널에서 필수!)
spark.sql("SELECT * FROM playground.sample_table.snapshots") \
    .show(truncate=False, vertical=True)

# [출력 결과에서 챙겨야 할 것]
# snapshot_id를 확인하세요! (예: 918297713420570779)
# 이 ID가 바로 우리가 돌아갈 '타임머신 목적지'입니다.
```


### 7. 시간 여행 (Time Travel / Rollback)

**Iceberg의 강력함 2:** 2차 입력이 잘못되었다고 가정하고, 1차 입력 직후로 되돌립니다.

- **기본 문법:** `{python}CALL [카탈로그명].system.[함수명]('테이블명', [인자값])`
	- **`system`**: Iceberg가 제공하는 시스템 관리 도구를 부르겠다는 뜻입니다.
	- **`rollback_to_snapshot`**: "이 스냅샷으로 되돌려라"라는 함수 이름입니다.

```python
# CALL 명령어: Spark SQL에서 시스템 함수를 호출할 때 씁니다.
# "playground.sample_table을 저 스냅샷 ID 상태로 롤백해줘"
# (실습 시 본인의 터미널에 뜬 snapshot_id 숫자를 넣어야 합니다!)
spark.sql("CALL system.rollback_to_snapshot('playground.sample_table', 918297713420570779)")
```

#### 실행 결과 해석

명령어를 실행하면 아래와 같은 1줄짜리 **DataFrame**이 반환됩니다.

| **previous_snapshot_id** | **current_snapshot_id** |
| ------------------------ | ----------------------- |
| 23847... (원래 있던 곳)       | 91829... (이동한 목적지)      |

- **`previous_snapshot_id`**: 롤백하기 직전(가장 최신)의 상태 ID
-  **`current_snapshot_id`**: 롤백 완료 후, 현재 테이블이 바라보고 있는 과거의 ID

### 8. 최종 확인

데이터가 과거로 돌아갔는지 확인합니다.

```python
# id 4, 5 데이터는 사라지고 1, 2, 3만 남아있습니다.
# 하지만 컬럼(extra_info) 구조는 남아있습니다. (데이터 상태만 롤백됨)
spark.sql("SELECT * FROM playground.sample_table").show()
```

### 다시 미래로 이동하기

- `history` 명령어로 족보를 봅니다. 현재(`true`)가 아닌, 끊어진 역사(`false`) 중에서 내가 돌아가고 싶은 ID를 찾습니다.

```python
# 역사를 조회합니다.
park.sql("SELECT * FROM playground.sample_table.history").show(truncate=False)
```

**결과**

```text
+-----------------------+------------------+------------------+-------------------+
|made_current_at        |snapshot_id       |parent_id         |is_current_ancestor|
+-----------------------+------------------+------------------+-------------------+
|2026-02-05 05:41:03.225|918297713420570779|NULL              |true               |
|2026-02-05 05:47:18.637|238473521353522660|918297713420570779|false              |. <-- 주목!
|2026-02-05 05:55:12.496|918297713420570779|NULL              |true               |
+-----------------------+------------------+------------------+-------------------+
```

- `is_current_ancestor` (현재 조상 여부) 
	- **true:** 지금 내 족보에 속해 있다. (유효한 역사) , **True**인 곳으로 갈 때 👉 **`CALL system.rollback_to_snapshot(...)`**
	- **false:** 지금 내 족보에서 끊겼다. (버려진 역사), **False**인 곳을 살릴 때 👉 **`CALL system.cherrypick_snapshot(...)`**
- `made_current_at` : 데이터가 생성된 시간이기도 하지만, 정확히는 **"이 스냅샷이 테이블의 '현재(Current)' 상태로 임명된 시각"** 입니다.
- `snapshot_id` : Iceberg가 관리하는 특정 시점의 상태를 가리키는 **절대적인 주소**입니다. `rollback_to_snapshot`이나 `cherrypick_snapshot` 같은 타임머신 명령어를 쓸 때 **목적지 주소**로 사용합니다.
- `parent_id` : 바로 직전(Before)의 스냅샷 ID입니다.`NULL`은 "내가 시초다(First Ancestor)"라는 뜻입니다. 테이블을 처음 만들고 데이터를 처음 넣었을 때가 여기에 해당합니다.
	- 마지막 줄이 NULL인 이유 : 이게 바로 롤백입니다

### 체리픽으로 복구하기 (핵심)

찾아낸 ID(`238473...`)를 가지고 체리픽을 수행합니다.

```python
# 주의: rollback_to_snapshot이 아닙니다!
# 내 화면에서 찾은 '파란색 숫자(미래 ID)'를 넣으세요.
target_id = 238473521353522660 

spark.sql(f"CALL system.cherrypick_snapshot('playground.sample_table', {target_id})")
```

- **`cherrypick_snapshot`**: 특정 스냅샷의 **변경 사항(데이터 추가/삭제 등)** 만 쏙 뽑아서 현재 상태 뒤에 **다시 실행(Replay)** 하는 원리입니다.
- 실행 후 `SELECT` 해보면 데이터가 5개로 복구되어 있을 것입니다.
