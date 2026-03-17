---
aliases:
  - docker exec
  - docker logs
  - -it
  - bash
tags:
  - Docker
related:
  - "[[00_Docker_HomePage]]"
  - "[[Docker_Lifecycle_Commands]]"
  - "[[Docker_Image_vs_Container]]"
  - "[[Docker_Compose_Commands]]"
---

# Docker_Container_Interaction — 컨테이너 내부 접속 & 로그

## 한 줄 요약

```
실행 중인 컨테이너 안으로 들어가거나
컨테이너가 무슨 말을 하는지 로그로 확인하는 명령어
```

---

---

# ① docker exec — 실행 중인 컨테이너에 명령어 보내기

```
docker exec [옵션] 컨테이너명 명령어

실행 중인 컨테이너 안에서 명령어를 실행
```

```bash
# 컨테이너 안에서 명령어 한 번만 실행
docker exec my-postgres ps aux
docker exec my-kafka ls /opt/kafka/bin

# 컨테이너 안으로 직접 들어가기 (-it + bash)
docker exec -it my-postgres bash
docker exec -it my-kafka bash
```

---

---

# ② -it 플래그 — "나 직접 대화할게"

```
-i  (interactive)  표준 입력(stdin) 열어둠
                   내가 키보드로 입력을 보낼 수 있음
-t  (tty)          터미널(terminal) 할당
                   화면이 예쁘게 보임 (프롬프트, 컬러 등)

둘 다 써야 제대로 된 쉘 환경이 됨
-i 만: 입력은 되는데 화면이 깨짐
-t 만: 화면은 나오는데 입력이 안 됨
-it: 완전한 터미널 환경
```

```bash
# 컨테이너 안으로 들어가기
docker exec -it my-postgres bash

# 들어가면 이런 프롬프트가 뜸
root@a3f2c1b4d5e6:/#

# 여기서 일반 리눅스 명령어 사용 가능
ls /
cat /etc/os-release
psql -U train_user -d train_db

# 나가기
exit
```

---

---

# ③ bash vs sh — 어떤 쉘로 들어가나?

```
bash   Bourne Again Shell — 더 많은 기능 (자동완성, 히스토리 등)
sh     Bourne Shell — 기본 쉘, 가볍지만 기능 적음

이미지마다 bash 가 없는 경우 있음
→ bash 로 안 되면 sh 로 시도
```

```bash
# bash 로 시도
docker exec -it my-container bash

# bash 없으면 에러
# OCI runtime exec failed: exec: "bash": executable file not found

# sh 로 대체
docker exec -it my-container sh

# Alpine 리눅스 기반 이미지는 주로 sh 만 있음
# (용량 최소화를 위해 bash 제외)
```

```
어떤 쉘 쓸지 빠르게 확인:
  docker exec my-container which bash   # /bin/bash 나오면 bash 있음
  docker exec my-container which sh     # /bin/sh 나오면 sh 있음
```

>[참고] Alpine 이미지는 **'알파인 리눅스(Alpine Linux)'라는 아주 작고 가벼운 운영체제를 기반으로 만든 도커 이미지**
`postgres:13-alpine`처럼 이름 뒤에 **'-alpine(알파인)'** 이 붙은 이미지

---

---

# ④ docker logs — 컨테이너 로그 보기

```
컨테이너가 stdout / stderr 로 출력하는 내용을 보여줌
print() / logging 으로 찍은 내용이 여기 나옴
```

```bash
# 지금까지 쌓인 로그 전체 출력
docker logs my-kafka

# 마지막 N줄만
docker logs --tail 50 my-kafka
docker logs --tail 100 my-airflow

# 타임스탬프 포함
docker logs -t my-kafka

# 실시간으로 계속 보기 (-f, --follow)
docker logs -f my-kafka

# 타임스탬프 + 실시간
docker logs -tf my-kafka

# 특정 시각 이후 로그만
docker logs --since 2026-03-17T02:00:00 my-airflow
docker logs --since 30m my-kafka        # 최근 30분
docker logs --since 1h my-kafka         # 최근 1시간
```

---

---

# ⑤ -f 실시간 모드 — tail -f 와 동일

```
-f (--follow)
  로그를 계속 스트리밍
  새 로그가 생길 때마다 바로 출력
  Ctrl+C 로 종료 (컨테이너는 안 꺼짐)

언제 쓰나:
  Airflow DAG 실행 중 실시간 확인
  Kafka Producer 발행 중 모니터링
  Spark Consumer 스트리밍 상태 확인
  에러 발생 시점 즉시 포착
```

```bash
# Airflow 로그 실시간 보기
docker logs -f my-airflow

# Spark Consumer 실시간 확인
docker logs -f my-spark-master

# 실시간 + 마지막 20줄부터
docker logs -f --tail 20 my-kafka
```

---

---

# ⑥ 실전 패턴 모음

```bash
# PostgreSQL 직접 접속
docker exec -it hospital-postgres bash
# 안에서
psql -U hospital_user -d hospital_db
\dt         # 테이블 목록
\q          # psql 종료

# Kafka 토픽 관리
docker exec -it hospital-kafka bash
# 안에서
/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic er-realtime --from-beginning

# Airflow DAG 수동 실행 확인
docker exec -it hospital-airflow bash
# 안에서
airflow dags list
airflow tasks list hospital_dag

# 에러 났을 때 로그 확인 패턴
docker logs --tail 100 hospital-airflow | grep ERROR
docker logs --tail 100 hospital-kafka   | grep WARN
```

---

---

# ⑦ exec vs run 차이

```
docker exec  이미 실행 중인 컨테이너에 명령어 추가
docker run   새 컨테이너를 만들어서 명령어 실행

docker exec -it my-postgres bash   ← 실행 중인 postgres 컨테이너 안으로 진입
docker run -it postgres bash       ← 새 postgres 컨테이너를 만들어서 bash 실행
                                      (기존 데이터 없음, 별개 컨테이너)
```

---

---

# 명령어 한눈에 정리

| 명령어                          | 동작              | 비고           |
| ---------------------------- | --------------- | ------------ |
| `docker exec -it 이름 bash`    | 컨테이너 내부 bash 접속 | bash 없으면 sh  |
| `docker exec -it 이름 sh`      | 컨테이너 내부 sh 접속   | Alpine 이미지에서 |
| `docker exec 이름 명령어`         | 명령어 한 번만 실행     | -it 없이       |
| `docker logs 이름`             | 전체 로그 출력        |              |
| `docker logs -f 이름`          | 실시간 로그 스트리밍     | Ctrl+C 로 종료  |
| `docker logs --tail N 이름`    | 마지막 N줄만         |              |
| `docker logs -t 이름`          | 타임스탬프 포함        |              |
| `docker logs --since 30m 이름` | 최근 30분 로그       |              |