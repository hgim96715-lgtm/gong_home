---
aliases:
tags:
  - Kafka
  - MinIO
related:
  - "[[MinIO_Concept]]"
  - "[[Pandas_Read_Write]]"
---
## 아키텍처 개요: 왜 이 작업이 필요한가?

> **"카프카의 실시간 데이터를 모아서 영구 보존용 창고(MinIO/S3)에 쌓아보자!"**

- **Kafka (고속도로):** 데이터가 빠르게 흘러가는 초고속 컨베이어 벨트. 영원히 보관해주지 않으므로 누군가 건져내야 한다.
- **MinIO (무한 창고):** 데이터를 파일 형태로 평생 보관하는 거대한 Data Lake. 하지만 스스로 물건을 가져오지 못한다.
- **Kafka Consumer (택배 기사님):** 우리가 파이썬으로 짤 프로그램. 지나가는 데이터(로그)를 주워서 박스에 담고, 예쁘게 포장(압축)해서 MinIO 창고에 집어넣는 역할을 한다.

---
## 핵심 도구: Pandas의 S3 통역사 (`s3fs`)

Pandas의 `to_parquet` 함수는 내 컴퓨터(로컬) 하드디스크만 볼 수 있다. 
바다 건너 클라우드 창고(MinIO/S3)에 접근하려면 **`s3fs`라는 통역사(라이브러리)** 와 **출입증(`storage_options`)** 을 반드시 쥐여줘야 한다.

- **필수 설치:** `pip install s3fs pyarrow`
- **`storage_options` 핵심 요약표**

|**옵션명 (Key)**|**역할 (비유)**|**🐳 로컬 MinIO (현재 세팅)**|**☁️ 실제 AWS S3 (실무)**|
|---|---|---|---|
|**`key`**|접속 ID (사원증)|`MINIO_ROOT_USER` 값|발급받은 `Access Key ID`|
|**`secret`**|비밀번호|`MINIO_ROOT_PASSWORD` 값|발급받은 `Secret Access Key`|
|**`client_kwargs`**|목적지 주소 (내비게이션)|**`{'endpoint_url': 'http://localhost:9000'}`**|**작성 안 함 (알아서 찾아감)**|