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
  - "[[Docker_Concept_vs_VM]]"
  - "[[Docker_Image_Layers]]"
  - "[[Spark_Installation_Local]]"
  - "[[Python_Virtual_Env]]"
---
## 개념 한 줄 요약

**"서버를 설치하는 과정을 순서대로 적어놓은 '요리 레시피(설계도)'."**

* **Dockerfile:** 텍스트로 된 설계도 파일.
* **Docker Build:** 설계도를 보고 실제로 요리해서 **이미지(Image)** 를 만드는 과정.
* **Docker Image:** 요리가 완성된 상태 (붕어빵 틀). 이걸로 컨테이너를 찍어냄.

---

##  왜 필요한가? (Why)

**문제점:**
- "서버 1대 세팅하는 데 3시간 걸렸는데, 10대 더 늘려야 한대요..." (반복 작업의 지옥)
- "어제는 잘 됐는데 오늘은 에러가 나요. 제가 뭘 설치했는지 기억이 안 나요." (환경 불일치)

**해결책:**
- 설치 과정을 `Dockerfile`에 적어두면, 명령어 한 줄(`docker build`)로 **100% 똑같은 서버 환경**을 1분 만에 만들 수 있습니다.
- 이것을 **IaC (Infrastructure as Code, 코드로 관리하는 인프라)** 라고 합니다.

---
##  핵심 명령어 (Keywords) 

스파크 실습 때 썼던 코드를 한 줄씩 뜯어봅시다.

```dockerfile
# 1. 베이스 이미지 (기초 공사)
# "어떤 OS, 어떤 프로그램 위에서 시작할래?"
FROM apache/spark:3.5.1

# 2. 사용자 변경 (권한)
# "이제부터 관리자(root) 권한으로 명령할게."
USER root

# 3. 파일 복사 (재료 준비) ️
# "내 컴퓨터(Host)에 있는 파일을 이미지 내부(Container)로 복사해."
# 문법: COPY [내컴퓨터경로] [이미지경로]
COPY requirements.txt /tmp/requirements.txt

# 4. 명령어 실행 (요리하기) ️
# "리눅스 명령어를 실행해서 프로그램을 설치해."
RUN pip install --no-cache-dir -r /tmp/requirements.txt
RUN pip install jupyterlab

# 5. 작업 디렉토리 설정 (이동)
# "이제부터는 /workspace 폴더 안에서 작업할게." (== cd /workspace)
WORKDIR /workspace

# 6. 포트 개방 (문 열기)
# "이 컨테이너는 8888번이랑 4040번 구멍을 쓸 거야." (문서화 용도)
EXPOSE 8888 4040

# 7. 실행 명령 (서빙)
# "컨테이너가 켜질 때 기본적으로 이 명령어를 실행해."
# (보통 CMD나 ENTRYPOINT를 씀)
CMD ["jupyter", "lab", "--ip=0.0.0.0"]
```

|옵션|의미|
|---|---|
|`-r`|패키지 목록 파일을 읽어라|
|`--no-cache-dir`|캐시 저장하지 마라|
> 더 , 자세한 내용은 [[Python_Virtual_Env#6. 심화 실무에서 쓰는 pip 옵션 꿀팁 (`-r`, `--no-cache-dir`)|pip 옵션]] 참고 
---
## 빌드 과정의 비밀: 레이어(Layer)와 캐시 

도커는 명령어를 한 줄 실행할 때마다 **임시 저장(Layer)** 을 합니다.

### ① 레이어 캐싱 (Layer Caching)

- **상황:** `requirements.txt`는 그대로인데, 소스 코드 한 줄만 고치고 다시 빌드함.
    
- **도커의 행동:**
    - `FROM ...` (변한 거 없네? 패스!)
    - `COPY requirements.txt ...` (변한 거 없네? 패스! - **캐시 사용**)
    - `RUN pip install ...` (위에서 안 변했으니 나도 결과 재탕!)
    - `COPY source_code ...` (**어? 이건 변했네? 여기서부터 다시 빌드!**)

- **결론:** **잘 안 바뀌는 명령어(라이브러리 설치 등)를 위쪽에**, **자주 바뀌는 명령어(소스 코드 복사)를 아래쪽**에 둬야 빌드가 엄청 빨라집니다.

>"`Dockerfile`을 짤 때는 **'순서'** 가 생명이야.
> 무거운 설치 작업(`pip install`)은 최대한 위로 올리고, 
> 자주 수정하는 코드 복사(`COPY . .`)는 맨 밑으로 내려. 
> 그래야 네가 코드 한 줄 고칠 때마다 5분씩 기다리는 불상사를 막을 수 있어!"

---
## 실전 명령어 (Build & Run)

```bash
# 1. 이미지 만들기 (Build)
# -t: 태그(이름) 붙이기
# . : 현재 폴더에 있는 Dockerfile을 써라
docker build -t my-spark-image:v1 .

# 2. 컨테이너 실행하기 (Run)
docker run -p 8888:8888 my-spark-image:v1
```

---
## 초보자가 자주 하는 실수

### ① `RUN` vs `CMD` 차이가 뭔가요?

- **`RUN`:** **이미지를 만드는 도중(요리 중)**에 실행됨. (예: `pip install`, `apt update`)
- **`CMD`:** **컨테이너가 시작될 때(서빙)** 딱 한 번 실행됨. (예: `python main.py`)

### ② `COPY` vs `ADD` 차이가 뭔가요?

- **`COPY`:** 그냥 파일 복사. (이걸 쓰세요. 명확합니다.)
- **`ADD`:** 복사 + 압축 해제 + URL 다운로드 기능까지 있음. (너무 마법을 부려서 권장하지 않음.)

### ③ "파일을 수정했는데 컨테이너에 반영이 안 돼요!"

- `Dockerfile`을 수정했으면 반드시 **다시 빌드(`docker build`)** 를 해야 이미지가 갱신됩니다.
- (단, `docker-compose`에서 `volumes`로 연결한 파일은 빌드 없이 실시간 반영됨.)