---
aliases:
  - MinIO와 Spark 분리 이유
  - Docker Architecture
  - 폴더 구조 전략
  - Project Structure
tags:
  - Architecture
  - Docker
related:
  - "[[MinIO_Docker_Setup]]"
  - "[[Apache_Spark_Setup]]"
---
## 개념 한 줄 요약

우리의 데이터 엔지니어링 프로젝트는 **"Airflow(본부) + MinIO(장기) + Spark(용병)"** 라는 3가지 핵심 요소로 구성된다.
이들의 역할과 특성에 따라 **Docker Compose 파일을 합칠지, 나눌지**를 결정하는 전략 가이드다.

---
## Architecture Diagram (최종 폴더 구조)

```text
📂 apache-airflow/ (메인 본부)
│  ├── docker-compose.yaml 
│  │   ├── 1. Airflow (Webserver, Scheduler)
│  │   ├── 2. Postgres (메타 DB)
│  │   ├── 3. Redis (메시지 브로커)
│  │   └── 4. MinIO (⭐️ 저장소 - 얘는 여기서 같이 사는 게 맞음!)
│  │
│  └── spark/ (용병 캠프 - 별도 관리)
│      ├── docker-compose.yaml
│      │   ├── 1. Spark Master
│      │   └── 2. Spark Worker
```

---
## Why MinIO is Merged? (합치는 이유)

#### MinIO = Airflow의 "장기" (Infrastructure) 

MinIO는 Airflow가 숨 쉬고 활동하기 위한 **"필수 저장소"** 야. 떼려야 뗄 수 없는 관계지.
1. **Life Cycle (생명주기):** Airflow가 켜질 때 MinIO도 **무조건** 같이 켜져 있어야 에러가 안 나. (로그 저장소 역할)
2. **Convenience (편의성):** `docker-compose up` 한 방에 Airflow와 MinIO가 같이 켜지는 게 훨씬 자연스러워.
3. **Networking (통신):** 같은 파일 안에 있으면 별도 설정 없이 `http://minio:9000`으로 바로 서로를 찾을 수 있어.

---
## Why Spark is Separated? (나누는 이유)

#### **Spark = Airflow가 부리는 "용병(Mercenary)" (External Compute)**

Spark는 Airflow의 신체 일부가 아니라, 가끔 무거운 짐을 처리해 주는 **"외부 일꾼"** 일 뿐이야.

1. **Independence (독립성):** Airflow가 죽어도 Spark 혼자서 분석 작업을 할 수도 있고, 반대로 Spark 없이 Airflow만 돌릴 때도 있어.
2. **Resource Heavy (리소스 돼지):** Spark는 메모리를 엄청나게 잡아먹어. Airflow 켤 때마다 무거운 Spark까지 항상 같이 켜지면 내 노트북(개발 환경)이 비명을 질러. 😱
3. **Management (관리):** 그래서 Spark는 **필요할 때만 `spark/` 폴더에서 따로 켜고**, 평소엔 꺼두는 게 효율적이야.

----
### 요약

- **MinIO:** **"내장(Internal)"** 취급 -> `docker-compose.yaml`에 포함
- **Spark:** **"외주(External)"** 취급 -> 별도 폴더로 격리.