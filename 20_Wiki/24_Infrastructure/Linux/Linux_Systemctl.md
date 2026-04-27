---
aliases:
  - systemctl
  - systemd
  - 데몬
  - daemon
  - service
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Boot_Process]]"
  - "[[Linux_Process]]"
  - "[[Linux_SSH]]"
---

# Linux_Systemctl — 데몬 & 서비스 관리

## 한 줄 요약

```
데몬(Daemon) = 백그라운드에서 묵묵히 돌아가는 서비스 프로세스
systemd      = 모든 데몬을 관리하는 PID 1 프로세스
systemctl    = systemd 를 제어하는 명령어
```

---

---

# ① 데몬(Daemon) 개념

```
데몬 = 사용자 개입 없이 백그라운드에서 계속 실행되는 프로세스

특징:
  터미널 없이 실행 (제어 터미널 없음)
  부팅 시 자동 시작
  이름이 d 로 끝나는 경우 많음

예시:
  sshd      → SSH 접속을 받는 데몬
  httpd     → 웹 요청을 처리하는 데몬 (Apache)
  nginx     → Nginx 웹서버 데몬
  postgresql → PostgreSQL 데몬
  crond     → crontab 실행 데몬
  dockerd   → Docker 데몬
```

## 왜 systemd 가 필요한가

```
부팅 시 수십 개의 데몬이 시작돼야 함
  네트워크 → SSH → DB → 웹서버 → ...

의존성 관리:
  DB 가 시작된 후에 웹서버가 시작돼야 함
  systemd 가 의존성 순서 자동 처리

병렬 시작:
  의존성 없는 서비스는 동시에 시작
  → 부팅 시간 단축
```

---

---

# ② systemctl 기본 명령어 ⭐️

## 서비스 상태 확인

```bash
# 서비스 상태 확인
sudo systemctl status nginx
sudo systemctl status ssh
sudo systemctl status postgresql

# q 로 종료
```

## 시작 / 중지 / 재시작

```bash
# 시작
sudo systemctl start nginx
sudo systemctl start ssh

# 중지
sudo systemctl stop nginx

# 재시작 (완전 중지 후 시작)
sudo systemctl restart nginx

# 설정 리로드 (재시작 없이 설정만 다시 읽기) ⭐️
sudo systemctl reload nginx
# nginx 는 reload 지원 (무중단)
# 지원 안 하는 서비스는 restart 필요
```

## 부팅 자동 시작 설정 ⭐️

```bash
# 부팅 시 자동 시작 등록
sudo systemctl enable nginx
# Created symlink /etc/systemd/system/... → ...

# 자동 시작 해제
sudo systemctl disable nginx

# 즉시 시작 + 부팅 자동 시작 동시에
sudo systemctl enable --now nginx

# 자동 시작 여부 확인
systemctl is-enabled nginx
# enabled  ← 자동 시작 O
# disabled ← 자동 시작 X
```

```
enable vs start:
  start  = 지금 당장 시작
  enable = 부팅 시 자동 시작 등록

  둘은 독립적:
  start 만 하면 → 재부팅하면 꺼짐
  enable 만 하면 → 지금은 안 켜짐, 다음 부팅부터 시작
  enable --now → 지금 켜고 + 부팅 시에도 자동 시작 ✅
```

---

---

# ③ systemctl status 출력 해석 ⭐️

```bash
sudo systemctl status ssh

# ● ssh.service - OpenBSD Secure Shell server
#      Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
#              ↑ 서비스 파일 위치   ↑ 부팅 자동 시작 여부
#      Active: active (running) since Mon 2026-04-20 10:00:00 UTC; 2h ago
#              ↑ 상태              ↑ 시작 시각
#    Main PID: 1234 (sshd)
#              ↑ 데몬 PID
#       Tasks: 1 (limit: 4915)
#      Memory: 5.4M
#      CGroup: /system.slice/ssh.service
#              └─1234 sshd: /usr/sbin/sshd -D
#
# Apr 20 10:00:01 hostname sshd[1234]: Server listening on 0.0.0.0 port 22.
# ↑ 최근 로그
```

## 상태 값 의미

|상태|의미|
|---|---|
|`active (running)`|정상 실행 중|
|`active (exited)`|실행 완료 (스크립트 등)|
|`inactive (dead)`|중지됨|
|`failed`|에러로 종료|
|`activating`|시작 중|
|`deactivating`|중지 중|

---

---

# ④ 전체 서비스 목록 확인

```bash
# 실행 중인 서비스만
systemctl list-units --type=service --state=running

# 전체 서비스 (실행 중 + 중지 포함)
systemctl list-units --type=service

# failed 상태 서비스만
systemctl list-units --state=failed

# 부팅 자동 시작 목록
systemctl list-unit-files --type=service
# nginx.service          enabled
# ssh.service            enabled
# bluetooth.service      disabled
```

---

---

# ⑤ 서비스 파일 — .service 유닛 ⭐️

```
서비스의 동작 방식을 정의하는 설정 파일
/lib/systemd/system/    → 패키지가 설치한 기본 파일
/etc/systemd/system/    → 사용자 커스텀 / override
```

## 서비스 파일 구조

```ini
# /lib/systemd/system/nginx.service

[Unit]
Description=A high performance web server and a reverse proxy server
After=network.target    # 네트워크 시작 후 실행

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t    # 시작 전 설정 검사
ExecStart=/usr/sbin/nginx          # 시작 명령어
ExecReload=/bin/kill -s HUP $MAINPID  # reload 명령
ExecStop=/bin/kill -s QUIT $MAINPID   # 중지 명령
Restart=on-failure                 # 실패 시 자동 재시작

[Install]
WantedBy=multi-user.target         # 어느 타겟에서 시작할지
```

## 서비스 파일 주요 옵션

```
[Unit]:
  Description  서비스 설명
  After        이 서비스들이 시작된 후 실행
  Requires     필수 의존 서비스
  Wants        선택적 의존 서비스

[Service]:
  Type         서비스 유형
    simple     기본 (ExecStart 가 메인 프로세스)
    forking    데몬이 fork 해서 백그라운드로
    oneshot    한 번 실행 후 종료
  ExecStart    시작 명령어
  ExecStop     중지 명령어
  ExecReload   reload 명령어
  Restart      재시작 조건
    on-failure  실패 시 재시작
    always      항상 재시작
  RestartSec   재시작 대기 시간 (초)
  User         실행 사용자

[Install]:
  WantedBy     활성화 시 어느 타겟에 연결
    multi-user.target  → CLI 부팅 시
    graphical.target   → GUI 부팅 시
```

## 커스텀 서비스 만들기

```bash
# 새 서비스 파일 생성
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Python App
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/myapp
ExecStart=/usr/bin/python3 /home/ubuntu/myapp/app.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# 새 서비스 파일 반영
sudo systemctl daemon-reload

# 등록 + 시작
sudo systemctl enable --now myapp
sudo systemctl status myapp
```

---

---

# ⑥ 서비스 로그 확인 — journalctl ⭐️

```bash
# 서비스 전체 로그
sudo journalctl -u nginx

# 실시간 로그 (tail -f 처럼)
sudo journalctl -u nginx -f

# 최근 N줄
sudo journalctl -u nginx -n 50

# 오늘 로그만
sudo journalctl -u nginx --since today

# 특정 시간 이후
sudo journalctl -u nginx --since "2026-04-20 10:00"

# 에러만
sudo journalctl -u nginx -p err

# 부팅 이후 로그
sudo journalctl -u nginx -b
```

---

---

# ⑦ service 명령어 (구버전)

```
SysV init 시절 명령어
systemctl 이전에 사용
여전히 많이 쓰임 (systemctl 의 wrapper)
```

```bash
# 상태 확인
service nginx status

# 시작 / 중지 / 재시작
service nginx start
service nginx stop
service nginx restart

# 실제로는 systemctl 을 호출함
# service nginx start = systemctl start nginx
```

---

---

# ⑧ 실전 패턴

## 데이터 엔지니어 자주 쓰는 서비스

```bash
# PostgreSQL
sudo systemctl start postgresql
sudo systemctl status postgresql
sudo systemctl enable postgresql  # 부팅 자동 시작

# Docker
sudo systemctl start docker
sudo systemctl enable docker

# SSH
sudo systemctl status ssh
sudo systemctl restart ssh

# Kafka (직접 설치한 경우)
sudo systemctl start kafka
sudo systemctl status kafka
```

## 서비스 failed 시 디버깅

```bash
# 1. 상태 확인
sudo systemctl status nginx

# 2. 로그 확인
sudo journalctl -u nginx -n 50

# 3. 설정 파일 문법 검사
sudo nginx -t              # nginx 예시
sudo sshd -t              # ssh 예시

# 4. 재시작 시도
sudo systemctl restart nginx

# 5. failed 서비스 초기화
sudo systemctl reset-failed nginx
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`systemctl status 서비스`|상태 확인|
|`systemctl start 서비스`|즉시 시작|
|`systemctl stop 서비스`|중지|
|`systemctl restart 서비스`|재시작|
|`systemctl reload 서비스`|설정만 리로드 (무중단)|
|`systemctl enable 서비스`|부팅 자동 시작 등록|
|`systemctl disable 서비스`|부팅 자동 시작 해제|
|`systemctl enable --now 서비스`|즉시 시작 + 자동 시작 동시 ⭐️|
|`systemctl is-enabled 서비스`|자동 시작 여부 확인|
|`systemctl list-units --type=service`|전체 서비스 목록|
|`systemctl list-units --state=failed`|실패한 서비스만|
|`systemctl daemon-reload`|서비스 파일 변경 후 갱신|
|`journalctl -u 서비스 -f`|실시간 로그|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|재부팅 후 서비스 꺼짐|enable 안 함|`systemctl enable --now`|
|서비스 파일 수정 후 반영 안 됨|daemon-reload 안 함|`systemctl daemon-reload`|
|reload 실패|서비스가 reload 미지원|`restart` 사용|
|status 에서 q 안 누름|less 뷰어 종료 몰름|`q` 로 종료|
|서비스 failed 상태 계속|에러 해결 후 reset 필요|`systemctl reset-failed`|