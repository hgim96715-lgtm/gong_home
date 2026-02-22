---
aliases:
  - MinIO
  - 미니오
  - Object Storage
  - S3 Compatible
  - S3 대체
tags:
  - Infrastructure
  - Storage
  - Docker
  - Tool
related:
  - "[[00_Airflow_HomePage]]"
  - "[[00_Kafka_HomePage]]"
  - "[[Kafka_Python_Consumer_Basic]]"
---
## 개념 한 줄 요약

**MinIO**는 **고성능(High-Performance)** 의 **분산 오브젝트 스토리지(Distributed Object Storage)** 시스템이야.
쉽게 말해, **"내 컴퓨터나 사내 서버에 설치해서 쓰는 공짜 AWS S3"** 라고 생각하면 돼.

---
## Why is it needed in Airflow? (핵심 이유)

우리가 Airflow 실습할 때 매번 AWS S3를 쓰면 **비용(돈)** 이 들잖아?
MinIO는 **Amazon S3 API와 완벽하게 호환(Fully Compatible)** 되기 때문에,
코드에서 `s3://`를 `minio://`로만 바꾸면 **돈 한 푼 안 들이고 S3 연동 실습**을 할 수 있어.

* **Log 저장:** Airflow 태스크 로그를 원격 저장소에 쌓을 때.
* **Data Lake:** 엑셀, CSV, Parquet 같은 비정형 데이터를 저장할 때.

---
## Key Features (주요 특징)

### ① S3 호환성 (S3 Compatibility) 
️
* Amazon S3용으로 만든 애플리케이션을 **코드 수정 없이** 그대로 MinIO에 붙일 수 있어.
* Airflow의 `S3Hook`이나 `S3Operator`를 그대로 사용 가능함.

### ② 고성능 & 확장성 (Performance & Scalability)

* **High Performance:** 빅데이터 분석이나 머신러닝 같은 고부하 작업에 최적화됨 (높은 처리량, 낮은 지연 시간).
* **Scalability:** 페타바이트(Petabytes) 단위까지 서버(노드)를 늘려서 수평 확장 가능.

### ③ 데이터 보호 & 보안 (Protection & Security)

* **Erasure Coding:** 데이터가 깨지거나 유실되지 않도록 보호하는 기술 지원.
* **Security:** 암호화(Encryption), TLS, 접근 권한 관리 등 엔터프라이즈급 보안 기능 제공.

### ④ 유연한 배포 (Deployment)

* 온프레미스(내 서버), 클라우드, **Kubernetes**, **Docker** 등 어디든 설치 가능.
* 👉 우리는 **Docker Compose**에 한 줄 추가해서 띄울 거야!

---
##  Technical Aspects (기술적 특징)

* **Go 언어 기반:** 구글이 만든 **Go 언어**로 작성되어 메모리를 적게 먹고 동시성 처리가 매우 효율적임.
* **오픈소스:** 기본 코어는 무료(Open-source)지만, 기업용 기술 지원은 유료로 구매 가능.

## Use Cases (어디에 써?)

1.  **Private Cloud Storage:** 우리 회사만의 S3를 구축하고 싶을 때.
2.  **Data Lakes:** 정형/비정형 대용량 데이터를 분석용으로 저장할 때.
3.  **Backup & DR:** 백업 및 재해 복구용 저장소.

---
## 아키텍처 관점: MinIO는 어떻게 데이터를 모을까? (vs Database)

"DB가 예쁘게 정리된 엑셀 진열대라면, MinIO는 무한대의 거대한 물류 창고다!"

### ① PostgreSQL(DB) vs MinIO(Data Lake)의 차이

- **PostgreSQL (DB):** 규칙에 맞게 잘 정리된 **'엑셀 표(진열대)'**. "12시 패션 매출 50만 원"처럼 연산이 끝난 요약/집계 데이터를 저장해 대시보드(Streamlit)에 빠르게 띄울 때 쓴다. 모든 원본 데이터를 넣으면 DB가 느려지고 터짐.

- **MinIO (Data Lake):** 규칙 없이 뭐든 다 때려 넣을 수 있는 **'초대형 무한 창고'**. 원본 로그 1,000만 건을 가공 없이 파일(Parquet, CSV, JSON 등) 형태로 안전하고 값싸게 쟁여두는 곳

### ② 왜 굳이 'Kafka Consumer' 코드를 짜서 데이터를 넣어야 할까?

초보자들이 가장 많이 헷갈리는 부분! **"창고(MinIO)는 입만 벌리고 있을 뿐, 스스로 물건을 가져오지 않는다"** 는 사실이다.

- **Kafka (카프카):** 물건(데이터)이 쉴 새 없이 지나가는 **'초고속 컨베이어 벨트'**. (시간이 지나면 데이터는 사라짐
- **MinIO (미니오):** 물건을 평생 쌓아둘 수 있는 **'거대 창고'**. 하지만 스스로 벨트에서 물건을 집어오지 못함.
- **Kafka Consumer (파이썬 적재 코드):** 컨베이어 벨트와 창고 사이에서 일하는 **'택배 기사님'**. 👉 기사님이 벨트(Kafka)에서 지나가는 박스 100개를 주워서 ➡️ 하나의 큰 박스로 꽁꽁 포장(압축)한 뒤(`to_parquet`) ➡️ 창고(MinIO)에 쑥 집어넣고 오는 역할을 한다!


---
### 💡 한마디

Airflow나 Kafka 파이프라인을 공부하는 입장에서 MinIO는 **"무료 실습용 S3 창고"** 라고 정의하면 가장 편합니다. Docker에 MinIO를 띄우면 `localhost:9000` 접속 주소가 생기는데, 이게 곧 AWS의 `s3.amazonaws.com` 역할을 하게 됩니다. 데이터가 빠르게 흘러가는 카프카(고속도로) 옆에, 데이터를 평생 보관할 수 있는 미니오(저수지)를 지어두면 실무와 똑같은 **'람다 아키텍처(Lambda Architecture)'** 가 완성됩니다!