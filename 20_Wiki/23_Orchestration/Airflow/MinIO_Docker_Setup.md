---
aliases:
  - MinIO 설치
  - MinIO Docker
  - docker-compose setup
  - S3 로컬 구축
tags:
  - Docker
  - MinIO
  - Setup
related:
  - "[[MinIO_Concept]]"
  - "[[00_Airflow_HomePage]]"
---
## 개념 한 줄 요약

Airflow가 실행되는 `docker-compose.yaml` 파일에 **MinIO 컨테이너를 추가**해서, 로컬 환경에 S3와 동일한 오브젝트 스토리지를 구축하는 가이드야.

---
## Code Core Points (docker-compose.yaml) 

기존 `docker-compose.yaml` 파일의 `services:` 섹션 아래에 이 코드를 붙여넣어.
(사용자가 지정한 **고정 버전**과 **경로**가 반영된 설정이야!)

```yaml
  # ------------------------------------
  # MinIO (S3 Compatible Storage)
  # ------------------------------------
  minio:
    image: minio/minio:RELEASE.2024-10-13T13-34-11Z  # 1. 버전 고정 (안정성 UP)
    container_name: minio
    hostname: minio
    restart: always             # 2. 죽어도 다시 살아남
    volumes:
      - ./include/data/minio:/data  # 3. 데이터 저장 위치 (사용자 지정 경로)
    ports:
      - "9000:9000"   # API Port (기계용)
      - "9001:9001"   # Console Port (사람용)
    environment:
      MINIO_ROOT_USER: ROOTNAME        # 4. ID (Access Key)
      MINIO_ROOT_PASSWORD: CHANGEME123 # 5. PW (Secret Key)
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
      # 나의 docker-compose에 networks가 없다면 설정안해줘도 됨 , 기본 default
```

---
## Detailed Analysis (설정값 해부)

### ① 이미지 버전 고정 (Image Pinning)

- `latest`를 쓰지 않고 날짜가 박힌 버전을 썼어.
- **Why?** MinIO가 갑자기 업데이트돼서 설정법이 바뀌더라도 내 시스템은 영향을 받지 않게 하기 위함이야. (프로덕션 레벨의 습관!)

### ② 볼륨 경로 (Volume Path)

- `- ./include/data/minio:/data`
- 컨테이너 안의 `/data` 폴더를 내 컴퓨터의 `./include/data/minio` 폴더와 연결했어.
- Docker를 삭제해도 파일은 `include` 폴더 안에 안전하게 남아있게 돼.

### ③ 계정 정보 (Credentials)

- **ID:** `ROOTNAME`
- **PW:** `CHANGEME123`
- 나중에 Airflow Connection 설정할 때 **이 값을 그대로 써야 하니까** 잘 기억해둬!

### ④ 헬스체크 (Healthcheck) 🏥

- `curl ... health/live`: Docker가 30초마다 "너 살아있니?" 하고 찔러봄.
- 응답이 없으면 문제가 있다고 판단해서(Unhealthy), `restart: always` 옵션 덕분에 자동으로 재시작을 시도해.

### ⑤ 네트워크 (Automatic Network)

- `docker-compose.yaml`에 `networks` 섹션이 없다면 Docker가 알아서 다 연결해 줘.
- MinIO 서비스에도 `networks`를 안 적으면, 자동으로 Airflow와 친구가 돼서 통신 가능해.

---
## Execution & Verification (실행 및 확인)

### Step 1. 실행

```bash
docker-compose up -d minio
```

### Step 2. 웹 접속 확인

- **주소:** `http://localhost:9001`
- **ID:** `ROOTNAME`
- **PW:** `CHANGEME123`
- 접속 성공 시 빨간색 MinIO 로고가 뜨면 성공!

### Step 3. 버킷 생성 (필수)

- 로그인 후 우측 하단 `+ Create Bucket` 클릭
- 버킷 이름: **`airflow-bucket`** (이 이름으로 실습할 예정)
---
## Deep Dive: 내 파일은 실제로 어떻게 저장될까? 

MinIO 웹(UI)에서 파일을 올리고 나서, 내 컴퓨터의 로컬 폴더(`include/data/minio/...`)를 열어보면 당황할 수 있어.
`input.txt` 파일은 안 보이고, 이상한 폴더와 파일이 있거든.

### ① 현상 (Observation)

* **상황:** 웹에서 `airflow-bucket` 버킷에 `input.txt`를 업로드함.
* **내 컴퓨터 확인:** `airflow-bucket/input.txt/` 라는 **폴더**가 생기고, 그 안에 **`xl.meta`** 라는 파일이 있음.

### ② 원인 (Why?)

MinIO는 단순한 파일 서버(FTP)가 아니라, **오브젝트 스토리지(Object Storage)** 야.
데이터의 무결성(깨짐 방지)과 버전 관리, 메타데이터 저장을 위해 파일을 통째로 변환해서 저장해.

* **`xl.meta`:** 파일의 정보(누가 올렸는지, 타입이 뭔지)와 **실제 데이터(Binary)** 가 합쳐진 MinIO만의 포맷이야.
* **폴더 구조:** MinIO는 객체 이름(`input.txt`)을 폴더 이름으로 사용해서 관리해.

### 🚨 주의사항 (Warning)

**"절대 로컬 폴더에서 `xl.meta` 파일을 직접 수정하거나 지우지 마!"**
* 내 컴퓨터 폴더에 있다고 해서 메모장으로 열어서 고치면, MinIO가 "어? 이거 내가 아는 파일 아닌데?" 하고 에러를 뱉거나 **데이터가 영구적으로 손상**될 수 있어.
* 파일 관리는 무조건 **MinIO 웹 UI**나 **API(파이썬, CLI)** 를 통해서만 해야 해.
---
## 초보자가 자주 착각하는 포인트

- **Q. "Airflow에서 연결할 때 호스트(Host) 이름을 뭘로 해요?"**
    - **A.** `hostname: minio`라고 적었으니, **`minio`** 가 곧 주소야.
    - `http://minio:9000` 으로 연결하면 돼. (`localhost` 아님! 컨테이너끼리는 이름으로 통신해)

- **Q. "네트워크 설정이 왜 중요한가요?"**
    - 만약 Airflow 컨테이너와 MinIO 컨테이너의 `networks:` 이름이 다르면, 서로 남남이라 통신이 안 돼. 
    - `docker-compose.yaml` 파일 맨 아래에 정의된 네트워크 이름과 맞춰줘.

- Q. 왜 docker-compose.yml에 왜 `networks`가 없어도 될까요?
	- Docker Compose는 `networks` 항목을 따로 적지 않으면, 자동으로 **`default`** 라는 하나의 네트워크(가상의 랜선)를 만들어서 **모든 컨테이너를 거기에 연결**시킵니다.
	- 파일에는 `airflow-worker`, `postgres` 등에도 `networks` 설정이 없다면,  MinIO도 똑같이 `networks` 설정을 안 적으면, **자동으로 Airflow와 같은 네트워크**에 묶이게 됩니다.
	- 따라서 서로 이름(`minio`, `postgres` 등)으로 통신하는 데 아무런 문제가 없습니다.
