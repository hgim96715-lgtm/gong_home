---
aliases:
  - Airflow 에러 모음
  - 트러블슈팅
  - AIRFLOW_UID
  - 설치 오류
tags:
  - Airflow
  - Error
  - Docker
  - Troubleshooting
related:
  - "[[00_Airflow_HomePage]]"
---
#  Common Airflow Errors (오답노트)

## Docker 설치 시: AIRFLOW_UID 설정 누락

> 참고 [[Airflow_Installation]]

### 개념 한 줄 요약

Docker 컨테이너(가상 환경)와 내 컴퓨터(Host) 간의 **'파일 주인(User ID)'** 이 서로 달라서 발생하는 권한 경고 및 실행 오류입니다.

### 왜 필요한가 (Why)

**문제 상황:**

* **에러 로그:** `{bash}WARN[0000] The "AIRFLOW_UID" variable is not set. Defaulting to a blank string.`
* **원인:** 도커 컨테이너는 기본적으로 `root`(관리자) 권한으로 파일을 생성하려고 합니다. 하지만 내 맥북의 주인은 나(`user`)입니다.
* **결과:** 도커가 만든 로그 파일을 내가 열어볼 수 없거나, 반대로 내가 만든 DAG 파일을 도커가 읽지 못하는 **'권한 거부(Permission Denied)'** 사태가 벌어집니다. 심하면 컨테이너가 켜지자마자 죽습니다.

**해결책:**
* Docker에게 **"야, 나(Host User)랑 똑같은 ID를 써!"** 라고 알려주는 환경 변수(`AIRFLOW_UID`)를 설정 파일(`.env`)에 적어줘야 합니다.

### 실무 맥락에서의 사용 이유

실무 리눅스 서버에서도 보안상의 이유로 Airflow를 절대 `root` 계정으로 실행하지 않습니다.
`airflow`라는 전용 계정을 만들어서 돌리는데, 이때도 Docker 컨테이너 안의 유저 ID와 서버의 `airflow` 계정 ID를 일치시켜주는 작업(UID 매핑)이 필수입니다. 이 에러는 **'컨테이너 보안 및 권한 관리'** 의 첫걸음입니다.

### 코드 핵심 포인트 (해결 명령어)

1.  **`id -u`**: 현재 내 컴퓨터의 유저 ID 번호를 알아냅니다. (보통 501번)
2.  **`.env`**: Docker Compose가 실행될 때 참고하는 **환경 변수 파일**입니다. 여기에 UID를 적어줍니다.

---
