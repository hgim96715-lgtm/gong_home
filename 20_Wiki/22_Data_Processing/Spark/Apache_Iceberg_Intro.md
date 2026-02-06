---
aliases:
  - 아파치 아이스버그
  - Iceberg
  - Table Format
  - Data Lakehouse
tags:
  - DataEngineering
  - Spark
  - DataLake
  - Architecture
related:
  - "[[Spark_Data_IO]]"
  - "[[Spark_Partitioning_Concept]]"
  - "[[Apache_Iceberg_Setup]]"
linked:
  - https://iceberg.apache.org/spark-quickstart/
---
## 개념 요약 

**Apache Iceberg**는 대용량 분석 데이터를 관리하기 위한 **오픈소스 테이블 포맷**입니다.
Netflix에서 처음 개발했으며, S3나 Azure Data Lake 같은 클라우드 스토리지 위에서 **ACID 트랜잭션**과 **SQL 기능**을 완벽하게 지원하도록 설계되었습니다.

> **💡 핵심 구분:**
> * **File Format:** Parquet, Avro, ORC (데이터를 어떻게 압축해서 저장할까?)
> * **Table Format:** **Iceberg** (수만 개의 파일들을 어떻게 하나의 '테이블'로 관리할까?)

---
##  Why: 기존 데이터 레이크(Hive)의 문제점과 해결책

* **문제점:**
    * **수정 불가:** 파일 하나를 고치려면 폴더 전체를 덮어써야 함.
    * **Schema 변경의 고통:** 컬럼 하나 추가하거나 이름 바꾸기가 매우 위험함 (데이터 꼬임).
    * **느린 성능:** 파일이 많아지면 S3에서 파일 목록(`ls`)을 가져오는 데만 한세월이 걸림.

>**기존의 문제:** 일반적인 파일(CSV, Parquet)로 데이터를 저장하면, 중간에 컬럼을 추가하거나 실수로 넣은 데이터를 되돌리기(Rollback)가 정말 어렵습니다.

* **해결책 (Iceberg):**
    * **ACID 트랜잭션:** 데이터 읽기/쓰기 시 **스냅샷 격리(Snapshot Isolation)** 를 보장하여, 쓰는 도중에 읽어도 깨진 데이터를 보지 않음.
    * **Schema Evolution:** 컬럼 추가/삭제/이름 변경이 데이터 파일 재작성 없이 즉시 가능함.

>즉,Iceberg를 사용하면 마치 데이터베이스처럼 테이블 구조를 자유롭게 바꾸고, 과거 시점으로 데이터를 복구할 수 있습니다.

---
## Key Features (핵심 무기)

### ① 스냅샷과 시간 여행 (Time Travel)

Iceberg는 테이블의 모든 변경 이력을 **스냅샷(Snapshot)** 으로 저장합니다.
* 실수로 데이터를 지웠어도, *"어제 14시 시점의 상태로 되돌려줘(Time Travel)"* 가 가능합니다.
* 변경된 데이터만 가져오는 **증분 읽기(Incremental Read)** 가 가능해 파이프라인이 단순해집니다.

### ② 숨겨진 파티셔닝 (Hidden Partitioning) ⭐
️
사용자가 물리적인 파티션 구조(예: `date=2024-01-01`)를 몰라도 됩니다.
* **기존:** 쿼리 짤 때 `WHERE date_str = '2024-01-01'` 처럼 파티션 컬럼을 정확히 맞춰야만 성능이 나옴.
* **Iceberg:** 그냥 `WHERE timestamp = '2024-01-01 10:00:00'` 라고 짜면, Iceberg가 알아서 해당 파티션을 찾아감. 물리적 저장 구조와 쿼리 로직을 분리함.

### ③ 다양한 엔진 지원 (Multi-Engine)

Spark뿐만 아니라 Trino, Flink, Hive, Presto 등 다양한 엔진에서 동시에 같은 테이블을 읽고 쓸 수 있습니다.

---
##  Architecture (어떻게 생겼나?)

Iceberg는 데이터를 3계층으로 관리합니다. (S3에 파일이 이렇게 쌓입니다) .

1.  **Catalog Layer:** "현재 최신 메타데이터 파일이 어디 있는지"를 가리키는 포인터.
2.  **Metadata Layer (The Brain):**
    * **Metadata File (`.json`):** 테이블의 스키마, 파티션 정보, 현재 스냅샷 ID 등을 저장.
    * **Manifest List (`.avro`):** 스냅샷을 구성하는 매니페스트 파일들의 목록.
    * **Manifest File (`.avro`):** 실제 데이터 파일(`parquet` 등)의 경로와 통계 정보(Min/Max 값 등)를 저장.
3.  **Data Layer:** 실제 데이터가 들어있는 **Parquet, Avro, ORC** 파일들.

---
##  Practical Context & Code

Iceberg는 테이블 뒤에 숨겨진 **메타데이터 테이블**을 조회해서 상태를 확인할 수 있습니다.

### 스냅샷 이력 확인하기 (`.snapshots`)

누가 언제 데이터를 썼는지(commit), 어떤 작업(append, overwrite)을 했는지 확인합니다.

```sql
-- 테이블명 뒤에 .snapshots를 붙여서 조회
SELECT 
    committed_at, 
    snapshot_id, 
    operation, 
    summary['added-records'] as added_rows
FROM 
    prod.db.my_table.snapshots;
```

### 타임라인 확인하기 (`.history`)

테이블의 변경 타임라인을 확인합니다.

```sql
SELECT * FROM prod.db.my_table.history;
```

---
## Tip

**언제 써야 하나요?**

- 데이터가 수시로 수정/삭제(Update/Delete) 되는 환경.
- 스키마가 자주 바뀌는 경우.
- "어제 데이터랑 비교해줘" 같은 시간 여행 쿼리가 필요할 때.

**주의점:**

- Iceberg도 결국 파일을 관리하는 것입니다. 
- 너무 자잘한 파일이 많아지면 주기적으로 **Compaction(파일 합치기)** 작업을 해줘야 성능이 유지됩니다.