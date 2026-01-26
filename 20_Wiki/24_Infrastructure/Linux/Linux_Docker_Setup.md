---
aliases:
  - 리눅스 실무 환경 구축
  - Docker 유저 권한 설정
tags:
  - Docker
  - Linux
  - Setup
related:
  - "[[Linux_Architecture]]"
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Fundamental_Rules]]"
  - "[[Filesystem Hierarchy Standard]]"
---
## 개념 한 줄 요약

실무 보안 가이드라인을 준수하여 **관리자(root)가 아닌 일반 사용자(ubuntu) 권한**으로 안전하게 동작하는 리눅스 실습 환경을 Docker로 구축하는 과정입니다.

---
## 왜 필요한가? (Why)

**보안 사고 방지:**
- 실무 환경에서 모든 프로세스를 `root` 권한으로 돌리는 것은 매우 위험합니다. 
- 해커가 서비스 하나만 뚫어도 서버 전체 권한을 넘겨주는 꼴이 되기 때문입니다. 
- 일반 유저를 기본으로 사용하는 습관을 들여야 합니다.

**권한 체계의 이해:**
- 데이터 엔지니어는 Airflow나 Spark를 운영할 때 파일 권한 문제(`Permission Denied`)를 자주 겪습니다.
- 처음부터 일반 유저 환경에서 실습해야 실제 현업에서 겪을 권한 이슈를 미리 체험하고 해결 능력을 키울 수 있습니다.

**멱등성 확보:**
- 내 맥북의 로컬 환경에 의존하지 않고, 팀원 모두가 동일한 리눅스 환경에서 작업할 수 있는 "표준화된 연습장"을 제공합니다.

---
## 실전 맥락 (Practical Context)

현업에서 Docker 이미지를 구울 때 **"Least Privilege(최소 권한 원칙)"** 를 지키는 것이 정석입니다. 
예를 들어, 기업용 Airflow 이미지는 내부적으로 `airflow`라는 유저를 따로 생성해 사용합니다. 
우리가 지금 `ubuntu` 유저를 생성하고 환경을 잡는 과정이 바로 **기업용 표준 이미지를 만드는 과정**과 동일합니다. 또한, `systemd`와 `SSH` 포트를 열어두어 원격 서버 관리 실습까지 가능하도록 설계되었습니다.

---
## Code Core Points (핵심 로직)

이 환경의 핵심은 **"빌드는 대장이 하고, 일은 일꾼이 하는 것"** 입니다.

**시스템 빌드 (Root):**
- 패키지 설치, 유저 생성, 시스템 스크립트 복사 등 집을 짓는 과정은 최고 관리자 권한으로 수행합니다.

**권한 위임:**
- 생성한 `ubuntu` 유저에게 `sudo` 권한을 부여해 필요할 때만 관리자 힘을 빌리게 합니다.

**문맥 전환 (USER):**
- 모든 공사가 끝나면 실행 주체를 `ubuntu` 유저로 완전히 전환하여 보안성을 높입니다.

---
## Detailed Analysis (상세 분석)

### ① Dockerfile (서버 설계도)

```Dockerfile
# 1. 우분투 22.04 LTS 버전을 베이스로 사용합니다.
FROM ubuntu:22.04

# 설치 중 인터랙티브 창이 뜨지 않게 설정합니다.
ARG DEBIAN_FRONTEND=noninteractive

# [기본 패키지 설치] - 이 작업은 root 권한으로 진행됩니다.
RUN apt-get update && apt-get install -y \
    openssh-server sudo systemd systemd-sysv cron \
    vim net-tools iputils-ping curl wget man-db plocate dumb-init \
    && apt-get clean

# [유저 생성 및 권한 부여] - 보안을 위해 실무 유저를 생성합니다.
RUN useradd -m -s /bin/bash ubuntu && \
    echo 'ubuntu:ubuntu' | chpasswd && \  # 유저 비밀번호 설정(비밀번호 ubuntu)
    adduser ubuntu sudo                   # ubuntu 유저를 sudo 그룹에 추가

# [시스템 설정] - SSH 통신을 위한 사전 준비
RUN mkdir /var/run/sshd
RUN echo 'root:root' | chpasswd

# [파일 복사 및 권한 설정] ⭐️ 중요! 
# 아직 USER 명령어를 쓰지 않았으므로 root 권한으로 파일을 복사하고 권한을 줍니다.
COPY bootstrap.sh /root/bootstrap.sh
RUN chmod +x /root/bootstrap.sh

# [볼륨 및 포트 설정]
VOLUME ["/sys/fs/cgroup"]
EXPOSE 22

# [유저 전환]  모든 인테리어가 끝난 뒤에 실제 거주자(ubuntu)로 바꿉니다.
USER ubuntu
WORKDIR /home/ubuntu

# [실행]
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["sudo", "/root/bootstrap.sh"] 
# 💡 팁: 서비스를 켜야 하는 bootstrap.sh는 sudo를 붙여 실행합니다.
```

### ② Docker Compose (실행 설정)

```YAML
services:
  linux:
    build: .
    container_name: linux-server
   # 시스템 제어 권한을 부여합니다. (systemd 등을 쓰기 위해 실습용으로만 허용)
    privileged: true
    ports:
      - "2222:22" # 외부(Mac)의 2222번 포트를 서버의 22번(SSH)과 연결
    volumes:
      - ./example:/example # 맥북의 ./example 폴더와 서버의 /example 폴더를 동기화
    tty: true # 터미널을 계속 켜두라는 명령
```

---
## 실행 방법 (How to Run)

이제 설계도(Dockerfile)와 시공 계획(compose)이 다 준비되었습니다.
터미널을 열고 서버를 켜볼까요?

### **1. 서버 생성 및 실행 (Background Mode)**

터미널에서 `docker-compose.yaml` 파일이 있는 폴더로 이동한 뒤 입력하세요.

```bash
docker-compose up -d --build
```

- `-d`: 백그라운드에서 실행(Detached)하라는 뜻입니다. 터미널을 계속 쓸 수 있게 해줍니다.
- `--build`: Dockerfile 내용이 바뀌었으면 새로 빌드하라는 뜻입니다.

### **2. 서버 내부로 접속 (Login)** 

서버가 켜졌다면, 이제 터미널 문을 열고 들어갑니다.

```bash
docker exec -it linux-server bash
```

- `exec`: 실행 중인 컨테이너에 명령을 내립니다.
- `-it`: 입출력이 가능한 터미널(Interactive TTY) 모드로 접속합니다.
- `bash`: 쉘 프로그램(Bash)을 실행합니다.

### 3. 접속 성공 확인

프롬프트가 `root@...`가 아닌 `ubuntu@...`로 떠야 성공입니다!

```bash
whoami
# 출력 결과: ubuntu
```

---
## 터미널 접속 메시지 해석 (Welcome Message)

- 처음에 시작할때 이렇게 나오는 메시지 해석 해보기 

```text
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```

### 의미 분석

- **"To run a command as administrator..."**: "관리자(대장) 권한으로 명령을 내리고 싶다면..."
	- 이 말이 떴다는 건, **현재 나는 관리자(root)가 아니다**라는 뜻입니다

### Docker 설정과의 연결 고리

아까 우리가 작성한 Dockerfile의 이 부분이 제대로 작동했다는 증거입니다.

```Dockerfile
# 이 설정 덕분에 리눅스가 나를 알아보고 인사를 건넨 것임!
RUN adduser ubuntu sudo
```

---
### 🔗 Next Step

축하합니다! 이제 나만의 리눅스 서버에 접속했습니다. 
하지만 막상 들어오니 `ubuntu@6baab...:~$` 같은 외계어가 보여 당황스러우신가요? 
이 낯선 기호들이 무엇을 의미하는지, **[[Linux_Fundamental_Rules]]** 에서 해석해 드립니다.

---
## 초보자가 자주 착각하는 포인트

- **"컨테이너를 지우면 내가 만든 파일도 사라지나요?"**
    - 네, 기본적으로는 사라집니다. 
    - 하지만 `volumes` 설정을 통해 내 맥북 폴더와 연결해둔 곳(예: `/example`)에 저장한 파일은 서버를 지워도 맥북에 안전하게 남습니다.
        
- **"왜 맥북 터미널을 안 쓰고 굳이 이걸 쓰나요?"**
    - 맥북은 BSD 계열 유닉스이고, 우리가 쓸 서버는 리눅스입니다.
    - `sed`, `awk` 같은 강력한 텍스트 처리 도구들의 옵션이 미세하게 달라서 리눅스에서 배우는 게 나중에 삽질을 안 하는 지름길입니다.

- **"맥북이랑 리눅스랑 명령어는 똑같지 않나요?"**
	- 비슷해 보이지만, 맥북은 BSD 계열이고 우리가 쓰는 우분투는 GNU 기반 리눅스입니다. 
	- 예를 들어 `sed -i` 옵션 하나만 써봐도 맥북에서는 에러가 나거나 문법이 다른 경우가 많습니다. 
	- 실무 환경과 100% 일치하는 곳에서 연습해야 나중에 실제 서버 배포 시 당황하지 않습니다.

- **"USER 명령어를 아무데나 넣어도 되나요?"**
	- **절대 안 됩니다.** 
	- `USER ubuntu` 명령어가 나오는 순간, 그 아래의 모든 명령(예: `chmod`, `mkdir`)은 일반 유저 권한으로 실행됩니다. 
	- 시스템 설정 파일이나 `/root` 폴더처럼 대장만 건드릴 수 있는 곳을 건드리려 하면 에러가 납니다. 
	- 공사가 다 끝나고 나서 집주인에게 열쇠를 넘겨줘야 합니다.

- **sudo를 썼는데 왜 비밀번호를 안 물어보나요?"**
	- Docker 빌드 과정이나 특정 설정에 따라 비밀번호 없이 실행되도록 세팅될 수 있습니다. 
	- 하지만 실제 운영 환경에서는 보안을 위해 비밀번호를 요구하는 것이 일반적입니다.


