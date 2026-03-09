---
aliases:
  - Dockerfile
  - 도커파일
  - ImageBuild
  - docker_build
  - 레이어
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Python_Virtual_Env]]"
  - "[[Docker_Image_Layers]]"
  - "[[Multi_Stage_Build]]"
  - "[[Docker_Compose_Setup]]"
---
# 🐳 Dockerfile_Basics

## 개념 한 줄 요약

> **"내 코드를 도커 이미지로 구워내는 레시피 파일."** 
> Dockerfile 은 이미지를 만드는 명령어들을 순서대로 적은 텍스트 파일이다. 
> `docker build` 명령어로 Dockerfile 을 읽어서 이미지를 생성한다.

```
Dockerfile  →  docker build  →  Image  →  docker run  →  Container
  레시피            굽기           완성품         실행            동작 중
```

---

---

# ① FROM — 베이스 이미지 지정

> **모든 Dockerfile 의 첫 줄. 어떤 이미지 위에 쌓을지 지정한다.**

```dockerfile
FROM python:3.11-slim
#      ↑ 이미지명  ↑ 태그 (버전)
```

```
태그를 안 쓰면 latest 가 자동 적용된다.
latest 는 버전이 바뀔 수 있어서 운영 환경에서는 반드시 버전 고정.

python:3.11-slim  → 경량화 버전 (용량 작음, 추천)
python:3.11       → 풀 버전 (용량 큼)
python:3.11-alpine → 초경량 (일부 라이브러리 호환 문제 있을 수 있음)
```

---

---

# ② WORKDIR — 작업 디렉토리 설정

> **이후 모든 명령어가 실행될 컨테이너 내부 경로를 지정한다.** 없으면 자동 생성. `cd` 랑 비슷하지만 영구적으로 적용.

```dockerfile
WORKDIR /app
```

```
WORKDIR 설정 전:   명령어가 루트(/) 에서 실행
WORKDIR /app 후:   이후 모든 RUN, COPY, CMD 가 /app 기준으로 실행

/app 폴더가 없어도 자동으로 생성해줌
```

---

---

# ③ COPY — 파일 복사

> **호스트(내 컴퓨터) 파일을 컨테이너 안으로 복사한다.**

```dockerfile
COPY requirements.txt .
#    ↑ 호스트 경로    ↑ 컨테이너 경로 (. = WORKDIR 기준 현재 위치)

COPY . .
# 현재 디렉토리 전체를 컨테이너의 WORKDIR 로 복사
```

```
⭐️ 레이어 캐시 최적화:
requirements.txt 를 먼저 복사하고 pip install 실행
그 다음에 소스코드 전체를 복사

코드만 바꿨을 때 pip install 레이어는 캐시 재사용 → 빌드 빠름
순서가 반대면 코드 한 줄 바꿀 때마다 pip install 을 다시 실행
```

```dockerfile
# ✅ 올바른 순서
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# ❌ 비효율적인 순서
COPY . .
RUN pip install -r requirements.txt
```

---

---

# ④ RUN — 빌드 시 명령어 실행

> **이미지를 만드는 과정에서 실행할 명령어. 패키지 설치, 파일 생성 등에 사용.** 실행 결과가 새로운 레이어로 저장된다.

```dockerfile
RUN pip install -r requirements.txt

# 여러 명령어는 && 로 이어서 레이어 수를 줄인다
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
#                                                 ↑ 캐시 삭제해서 용량 최소화
```

```
RUN 한 줄 = 레이어 1개
레이어가 많아질수록 이미지 용량 증가
→ && 로 이어서 레이어 수 최소화 권장
```

---

---

# ⑤ ENV — 환경변수 설정

> **컨테이너 실행 시 사용할 환경변수를 설정한다.** 코드에서 `os.environ['변수명']` 으로 읽을 수 있다.

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV APP_PORT=8080
```

```
PYTHONDONTWRITEBYTECODE=1  → .pyc 파일 생성 안 함 (컨테이너 불필요)
PYTHONUNBUFFERED=1         → 로그를 버퍼링 없이 즉시 출력 (docker logs 에서 바로 보임)
```

> `.env` 파일의 값은 `docker-compose.yml` 에서 주입. 비밀번호, API 키 같은 민감한 값은 Dockerfile 에 직접 쓰지 말 것.

---

---

# ⑥ EXPOSE — 포트 문서화

> **컨테이너가 사용할 포트를 선언한다.** 실제로 포트를 여는 것이 아니라 "이 포트를 쓸 거야" 라고 알려주는 문서화 역할.

```dockerfile
EXPOSE 8080
```

```
EXPOSE 8080  →  "이 컨테이너는 8080 포트를 씁니다" 라는 선언
실제 포트 연결은 docker run -p 또는 docker-compose.yml 의 ports 에서 설정

EXPOSE 없어도 포트 연결 가능
하지만 다른 사람이 봤을 때 어떤 포트를 쓰는지 알 수 있도록 적는 것이 관례
```

---

---

# ⑦ CMD vs ENTRYPOINT — 컨테이너 실행 명령어

> **컨테이너가 시작될 때 실행할 명령어.** 둘 다 비슷해 보이지만 역할이 다르다.

## CMD — 기본 명령어 (덮어쓰기 가능)

```dockerfile
CMD ["python", "app.py"]
```

```
컨테이너 실행 시 기본으로 실행할 명령어
docker run 시 뒤에 명령어를 붙이면 CMD 가 무시됨

docker run myimage                  → python app.py 실행
docker run myimage python test.py   → test.py 실행 (CMD 무시됨)
```

## ENTRYPOINT — 고정 명령어 (덮어쓰기 불가)

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

```
ENTRYPOINT 는 항상 실행됨 (덮어쓰기 불가)
CMD 는 ENTRYPOINT 의 기본 인자 역할

docker run myimage           → python app.py
docker run myimage test.py   → python test.py (CMD 만 바뀜)
```

## 비교

| |CMD|ENTRYPOINT|
|---|---|---|
|**역할**|기본 실행 명령어|고정 실행 명령어|
|**덮어쓰기**|`docker run` 뒤에 명령어로 가능|`--entrypoint` 옵션으로만 가능|
|**주로 사용**|실행 파일 지정|특정 실행 파일을 강제할 때|

```
일반적인 Python 앱:    CMD ["python", "app.py"]
항상 python 으로 실행: ENTRYPOINT ["python"] + CMD ["app.py"]
```

---

---

# 전체 예제 — Python 앱 Dockerfile

```dockerfile
# ① 베이스 이미지
FROM python:3.11-slim

# ② 작업 디렉토리 설정
WORKDIR /app

# ③ 환경변수
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# ④ 의존성 먼저 복사 + 설치 (캐시 최적화)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ⑤ 소스코드 복사
COPY . .

# ⑥ 포트 선언
EXPOSE 8080

# ⑦ 실행 명령어
CMD ["python", "app.py"]
```

---

---

# 빌드 & 실행

```bash
# 이미지 빌드
$ docker build -t my-app:1.0 .
#               ↑ 이름:태그    ↑ Dockerfile 위치 (현재 디렉토리)

# 컨테이너 실행
$ docker run -d -p 8080:8080 --name my-container my-app:1.0
#            ↑   ↑            ↑                   ↑
#         백그라운드 포트연결  컨테이너이름          이미지이름
```

---

# 초보자 실수 체크리스트

|실수|원인|해결|
|---|---|---|
|`FROM` 태그 없이 `latest` 사용|버전이 바뀌면 빌드 결과가 달라짐|버전 명시 `python:3.11-slim`|
|`COPY . .` 를 `RUN pip install` 앞에 배치|코드 수정마다 pip install 재실행|`requirements.txt` 먼저 COPY|
|`RUN` 을 여러 줄로 분리|레이어 수 증가 → 이미지 용량 증가|`&&` 로 이어서 작성|
|비밀번호를 `ENV` 에 하드코딩|이미지에 비밀번호가 박혀버림|`.env` + `docker-compose` 로 주입|
|`CMD` 와 `ENTRYPOINT` 혼동|의도치 않게 명령어가 덮어써짐|역할 구분 후 상황에 맞게 사용|

---

