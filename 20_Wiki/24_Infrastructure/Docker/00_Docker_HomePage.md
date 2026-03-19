
```
"내 컴퓨터에서는 되는데 네 컴퓨터에서는 안 돼?"
이 지긋지긋한 말을 끝내는 기술

환경(Environment)을 통째로 포장해서 배달하는 컨테이너의 세계
```

---

---
## Level 0. 개념 잡기

```
가상머신(VM) 이랑 뭐가 달라요?
```

|노트|핵심 개념|
|---|---|
|[[Docker_Concept_vs_VM]]|VM은 집 전체 / Docker는 방 한 칸 / Hypervisor / 격리 / OS 공유 / 경량화|
|[[Docker_Architecture]]|Client / Daemon / Registry 관계 / Docker Hub / dockerd / 소켓 통신|
|[[Docker_Image_vs_Container]]|이미지(Program) vs 컨테이너(Process) / 읽기전용 vs 읽기쓰기 / 레이어 / Bitnami vs Apache 공식 이미지|

---

---

## Level 1. 컨테이너 조작하기

```
일단 남이 만든 거 가져와서 실행해보자
```

| 노트                                | 핵심 개념                                                      |
| --------------------------------- | ---------------------------------------------------------- |
| [[Docker_Lifecycle_Commands]] ⭐   | run / start / stop / rm / ps / -d 백그라운드 / --name           |
| [[Docker_Container_Interaction]]⭐ | exec -it / logs / -f 실시간 / bash / sh                       |
| [[Docker_Prune]]                  | system prune / image prune / volume prune / -a 전체 / 디스크 확보 |

---

---

## Level 2. 나만의 이미지 만들기 (Dockerfile)

```
내 코드를 Docker 이미지로 구워보자
```

| 노트                               | 핵심 개념                                                         |
| -------------------------------- | ------------------------------------------------------------- |
| [[Dockerfile_Basics]]            | FROM / RUN / COPY / CMD / ENTRYPOINT / WORKDIR / ENV / EXPOSE |
| [[Docker_Image_Layers]]          | 레이어 캐싱 원리 / 빌드 속도 최적화 / COPY 위치 / 캐시 무효화                      |
| [[Docker_Python_Requirements]] ⭐ | requirements.txt / pip freeze / pip install / 버전 고정 / 의존성     |

---

---

## Level 3. 저장소와 네트워크

```
컨테이너를 껐다 켜도 데이터가 살아있으려면?
```

| 노트                    | 핵심 개념                                                  |
| --------------------- | ------------------------------------------------------ |
| [[Docker_Volumes]]    | -v / volumes / bind mount / tmpfs / 영구 저장 / 호스트 경로 마운트 |
| [[Docker_Networking]] | --network / -p 포트포워딩 / bridge / host / 컨테이너 간 DNS      |

---

---

## Level 4. Docker Compose

```
컨테이너 여러 개를 한 번에 켜고 끄자
```

| 노트                            | 핵심 개념                                                                            |
| ----------------------------- | -------------------------------------------------------------------------------- |
| [[Docker_Compose_Setup]]      | docker-compose.yml 문법 / services / volumes / networks / depends_on / environment |
| [[Docker_Compose_Commands]] ⭐ | up / down / ps / logs / --build / -d 백그라운드                                       |

---

---

## Level 5. 실전 팁

|노트|핵심 개념|
|---|---|
|[[Docker_Permission_Handling]]|sudo 없이 도커 쓰기 / USER / docker group / chmod / uid/gid / 권한 에러|
|[[Multi_Stage_Build]]|이미지 1GB → 50MB / Multi-Stage / AS builder / COPY --from / 용량 최적화|

---

---

## 프로젝트 적용

| 노트                                   | 설명                           |
| ------------------------------------ | ---------------------------- |
| [[01_Docker_Setup_Postgresql_Setup]] | 서울역 프로젝트 Docker Compose      |
| [[01_Hospital_Docker_Setup]]         | Hospital 프로젝트 Docker Compose |
| [[PostgreSQL_Setup]] ⭐               | PostgreSQL + Docker 환경 세팅    |

---

---
## Templates

| 노트                          | 설명                                                                              |
| --------------------------- | ------------------------------------------------------------------------------- |
| [[Docker_Compose_Template]] | Kafka / PostgreSQL / Spark / Airflow / Superset 기본 구조 — PROJECT_NAME 교체해서 바로 사용 |
