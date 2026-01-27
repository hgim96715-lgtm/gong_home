

> **"내 컴퓨터에서는 되는데 네 컴퓨터에서는 안 돼?"**
> 이 지긋지긋한 말을 끝내는 기술. 환경(Environment)을 통째로 포장해서 배달하는 **컨테이너(Container)** 의 세계입니다.


## Level 0. 개념 잡기 (Concept)

> "가상머신(VM)이랑 뭐가 달라요?"

- [ ] **[[Docker_Concept_vs_VM]]** : 가상머신은 '집 전체'를 빌리는 것, 도커는 '방 한 칸'만 빌리는 것.
- [ ] **[[Docker_Architecture]]** : Client, Daemon, Registry의 관계.
- [ ] **[[Image_vs_Container]]** : 프로그램(Image)과 프로세스(Container)의 차이.

## Level 1. 컨테이너 조작하기 (Basic Commands)

> "일단 남이 만든 거 가져와서 실행해보자."

- [ ] **[[Docker_Lifecycle_Commands]]** : 생성, 시작, 정지, 삭제 (`run`, `start`, `stop`, `rm`)
- [ ] **[[Docker_Container_Interaction]]** : 컨테이너 내부로 들어가기 & 로그 보기 (`exec`, `logs`)
- [ ] **[[Docker_Prune]]** : 쓰레기 청소하기 (`system prune`)

## Level 2. 나만의 이미지 만들기 (Dockerfile)

> "내 코드를 도커 이미지로 구워보자."

- [x] **[[Dockerfile_Basics]]** : 이미지를 만드는 핵심 명령어 (`FROM`, `RUN`, `COPY`, `CMD`)
- [ ] **[[Docker_Image_Layers]]** : 레이어(Layer) 캐싱의 원리와 빌드 속도 최적화.

## Level 3. 저장소와 네트워크 (Storage & Network) 

> "컨테이너를 껐다 켜도 데이터가 살아있으려면?"

- [ ] **[[Docker_Volumes]]** : 데이터를 영구 저장하는 법 (`-v`, `volumes`) 
- [ ] **[[Docker_Networking]]** : 컨테이너끼리 통신하는 법 (`--network`, 포트 포워딩 `-p`)

## Level 4. 오케스트레이션 (Docker Compose) 

> "컨테이너 3개를 한 번에 켜고 끄자."

- [ ] **[[Docker_Compose_Setup]]** : `docker-compose.yml` 문법 완벽 정리. **(스파크 클러스터 구축의 핵심)**
- [ ] **[[Compose_Commands]]** : 한 방에 실행하고 관리하기 (`up`, `down`, `ps`)
    

## Level 5. 실전 팁 (Advanced)

- [ ] **[[Docker_Permission_Handling]]** : `sudo` 없이 도커 쓰기 & 컨테이너 내부 유저 권한 (`USER`)
- [ ] **[[Multi_Stage_Build]]** : 이미지 용량을 1GB에서 50MB로 줄이는 마법.