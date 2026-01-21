---
aliases:
  - Docker localhost 접근
  - host.docker.internal
  - 도커 호스트 연결
  - DB 연결
tags:
  - Docker
  - Network
  - 트러블슈팅
related:
  - "[[Airflow_Installation]]"
---
## 개념 한 줄 요약

Docker 컨테이너 안에서 **내 컴퓨터(Mac/Windows)에 설치된 DB나 프로그램**에 접속하고 싶을 때, `localhost` 대신 **`host.docker.internal`** 을 사용해야 합니다.

---
## 왜 필요한가 (Why)

**문제 상황:**

* 내 맥북에 **PostgreSQL**을 깔아두고(포트 5432), Docker에 뜬 Airflow가 거기에 접속하게 하고 싶음.
* Airflow connection에 `localhost:5432`라고 적으면 에러가 남.
* **이유:** Docker 입장에서 `localhost`는 **'Docker 컨테이너 그 자체'** 이지, 내 맥북이 아니기 때문.

**해결책:**

* Docker가 만들어준 뒷문(Backdoor)인 **`host.docker.internal`** 주소를 사용하면, 컨테이너가 벽을 뚫고 내 맥북의 포트에 접근할 수 있음.

---
##  실무 적용 예시 (Connection String)

### ❌ 틀린 방법 (접속 불가)

```text
postgresql://my_user:my_password@localhost:5432/my_db
# 에러: Connection refused (컨테이너 안에는 5432 포트가 없으니까!)
```

### ✅ 맞는 방법 (접속 성공)

```text
postgresql://my_user:my_password@host.docker.internal:5432/my_db
# 성공: 내 맥북의 5432 포트로 연결됨
```

---
## 주의사항

- **Mac/Windows 사용자:** Docker Desktop을 쓴다면 기본적으로 활성화되어 있어서 그냥 쓰면 됩니다.
- **Linux 사용자:** `docker-compose.yaml` 파일에 `extra_hosts` 설정을 추가해야 할 수도 있습니다. (하지만 우리는 맥북이니까 패스!)

---
## 보자가 자주 착각하는 포인트

**"Providers 깔려면 Standalone이 낫지 않나요?"**

- **절대 아닙니다!**
- Providers를 깔려면 `pip install apache-airflow-providers-postgres` 같은 걸 해야 하는데, Standalone(로컬)에서 하면 내 맥북의 파이썬 버전, 라이브러리 버전이랑 꼬여서 **"의존성 지옥(Dependency Hell)"** 에 빠집니다.
- Docker에서는 `docker-compose.yaml`이나 `Dockerfile`에 한 줄만 추가하면 깔끔하게 설치되고, 맘에 안 들면 컨테이너를 지워버리면 그만입니다. 훨씬 안전합니다.