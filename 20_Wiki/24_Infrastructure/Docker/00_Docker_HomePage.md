# 🐳 Docker

> **"내 컴퓨터에서는 되는데 네 컴퓨터에서는 안 돼?"** 이 지긋지긋한 말을 끝내는 기술. 
> 환경(Environment)을 통째로 포장해서 배달하는 컨테이너(Container)의 세계.

---

## Level 0. 개념 잡기 (Concept)

> "가상머신(VM)이랑 뭐가 달라요?"

- [[Docker_Concept_vs_VM]] — 가상머신은 '집 전체'를 빌리는 것, 도커는 '방 한 칸'만 빌리는 것 `VM` `Hypervisor` `Container` `격리` `OS 공유` `경량화`
- [[Docker_Architecture]] — Client, Daemon, Registry 의 관계 `Docker Client` `Docker Daemon` `Docker Hub` `Registry` `dockerd` `소켓 통신`
- [[Image_vs_Container]] — 프로그램(Image)과 프로세스(Container)의 차이 `Image` `Container` `읽기전용 vs 읽기쓰기` `레이어` `스냅샷`

---

## Level 1. 컨테이너 조작하기 (Basic Commands)

> "일단 남이 만든 거 가져와서 실행해보자."

- [[Docker_Lifecycle_Commands]] — 생성, 시작, 정지, 삭제 `run` `start` `stop` `rm` `ps` `-d 백그라운드` `--name`
- [[Docker_Container_Interaction]] — 컨테이너 내부로 들어가기 & 로그 보기 `exec -it` `logs` `-f 실시간` `bash` `sh`
- [[Docker_Prune]] — 쓰레기 청소하기 `system prune` `image prune` `volume prune` `-a 전체` `디스크 확보`

---

## Level 2. 나만의 이미지 만들기 (Dockerfile)

> "내 코드를 도커 이미지로 구워보자."

- [[Dockerfile_Basics]] — 이미지를 만드는 핵심 명령어 `FROM` `RUN` `COPY` `CMD` `ENTRYPOINT` `WORKDIR` `ENV` `EXPOSE`
- [[Docker_Image_Layers]] — 레이어 캐싱의 원리와 빌드 속도 최적화 `Layer` `캐시 히트` `빌드 순서` `COPY 위치` `캐시 무효화`
- [[Docker_Python_Requirements]] — requirements.txt 작성법과 pip 관리 `requirements.txt` `pip freeze` `pip install` `버전 고정` `의존성`

---

## Level 3. 저장소와 네트워크 (Storage & Network)

> "컨테이너를 껐다 켜도 데이터가 살아있으려면?"

- [[Docker_Volumes]] — 데이터를 영구 저장하는 법 `-v` `volumes` `bind mount` `tmpfs` `영구 저장` `호스트 경로 마운트`
- [[Docker_Networking]] — 컨테이너끼리 통신하는 법 `--network` `-p 포트포워딩` `bridge` `host` `컨테이너 간 DNS` `같은 네트워크`

---

## Level 4. 오케스트레이션 (Docker Compose)

> "컨테이너 3개를 한 번에 켜고 끄자."

- [[Docker_Compose_Setup]] — `docker-compose.yml` 문법 완벽 정리 `services` `volumes` `networks` `depends_on` `environment`
- [[Docker_Compose_Commands]] — 한 방에 실행하고 관리하기 `up` `down` `ps` `logs` `--build` `-d 백그라운드`

---

## Level 5. 실전 팁 (Advanced)

- [[Docker_Permission_Handling]] — `sudo` 없이 도커 쓰기 & 컨테이너 내부 유저 권한 `USER` `docker group` `chmod` `uid/gid` `권한 에러 해결`
- [[Multi_Stage_Build]] — 이미지 용량을 1GB → 50MB 로 줄이는 마법 `Multi-Stage` `AS builder` `COPY --from` `용량 최적화` `빌드 도구 제거`

---

## 프로젝트 적용

- [[01_Docker_Setup_Postgresql_Setup]] — 실전 프로젝트에서 Docker Compose 적용