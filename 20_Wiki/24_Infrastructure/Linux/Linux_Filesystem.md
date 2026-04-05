---
aliases:
  - 리눅스 파일 시스템
  - 디렉토리 구조
  - 절대경로
  - 상대경로
tags:
  - Linux
related:
  - "[[Linux_Basic_Commands]]"
  - "[[Linux_File_Commands]]"
  - "[[Linux_File_Permissions]]"
  - "[[00_Linux_HomePage]]"
---

# Linux_Filesystem — 파일시스템 & 디렉토리 구조

## 한 줄 요약

```
리눅스는 "모든 것이 파일"
최상위 / (루트) 에서 시작하는 트리 구조
어디에 뭐가 있는지 알아야 트러블슈팅 가능
```

---

---

# ① 디렉토리 트리 전체 구조

```
/ (루트)
├── bin      → 기본 명령어 (ls, cp, mv 등)
├── sbin     → 시스템 관리 명령어 (root 용)
├── etc      → 설정 파일
├── home     → 일반 사용자 홈 디렉토리
├── root     → root 사용자 홈
├── var      → 가변 데이터 (로그, 캐시, DB 데이터)
├── tmp      → 임시 파일 (재부팅 시 삭제)
├── usr      → 사용자 프로그램 및 데이터
├── lib      → 공유 라이브러리
├── dev      → 장치 파일 (하드웨어를 파일로 표현)
├── proc     → 프로세스 정보 (가상 파일시스템)
├── sys      → 커널 / 하드웨어 정보
├── mnt      → 임시 마운트 포인트
├── media    → USB / CD 등 외부 미디어
├── opt      → 서드파티 소프트웨어
└── srv      → 서비스 데이터 (웹서버 등)
```

---

---

# ② 핵심 디렉토리 역할 ⭐️

## /etc — 설정 파일의 집

```
시스템 전체 설정 파일
수정하면 서비스 동작 방식이 바뀜
텍스트 파일 위주 → vim / nano 로 편집
```

```bash
/etc/passwd          # 사용자 계정 정보
/etc/shadow          # 암호화된 비밀번호
/etc/group           # 그룹 정보
/etc/hosts           # 호스트명 → IP 매핑
/etc/resolv.conf     # DNS 서버 설정
/etc/hostname        # 시스템 호스트명
/etc/fstab           # 파일시스템 마운트 설정
/etc/crontab         # 크론 스케줄

/etc/nginx/          # Nginx 설정
/etc/ssh/            # SSH 설정
/etc/apt/            # APT 패키지 관리 설정
```

## /var — 변하는 데이터

```
프로그램이 실행되면서 계속 변하는 데이터
로그 / 캐시 / 런타임 파일 / DB 데이터 등
용량이 커질 수 있어서 별도 파티션으로 많이 운영
```

```bash
/var/log/            # 로그 파일
  /var/log/syslog    # 시스템 로그
  /var/log/auth.log  # 인증 로그 (ssh 접속 등)
  /var/log/nginx/    # Nginx 로그

/var/lib/            # 애플리케이션 상태 데이터
  /var/lib/postgresql/  # PostgreSQL 데이터
  /var/lib/docker/      # Docker 이미지 / 컨테이너

/var/cache/          # 캐시 데이터
/var/tmp/            # 재부팅 후에도 유지되는 임시 파일
/var/run/            # 실행 중인 프로세스 PID 파일
```

## /home — 사용자 홈 디렉토리

```
일반 사용자 개인 파일 공간
/home/사용자명 형식
~  = 현재 사용자 홈 디렉토리 단축키
```

```bash
/home/gong/          # gong 사용자 홈
  ~/.bashrc          # bash 설정
  ~/.bash_profile    # 로그인 셸 설정
  ~/.ssh/            # SSH 키
  ~/.config/         # 앱별 설정

# root 사용자 홈은 별도
/root/               # root 홈 (~ 이 아닌 /root)
```

## /usr — 사용자 프로그램

```
Unix System Resources
읽기 전용 (설치된 프로그램과 데이터)
apt / yum 으로 설치한 것들이 여기에
```

```bash
/usr/bin/            # 일반 사용자용 실행 파일
  /usr/bin/python3   # Python
  /usr/bin/git       # Git

/usr/sbin/           # 시스템 관리용 실행 파일 (root)
/usr/lib/            # 라이브러리
/usr/local/          # 직접 컴파일 설치한 프로그램
  /usr/local/bin/    # 직접 설치한 실행 파일
/usr/share/          # 공유 데이터 (문서, 아이콘 등)
```

## /tmp — 임시 파일

```
재부팅하면 전부 삭제
모든 사용자가 쓸 수 있음 (권한 777)
스크립트 임시 파일 / 테스트용으로 활용
```

```bash
# 임시 파일 만들기
mktemp                    # /tmp/tmp.XXXXXX 형식으로 생성
mktemp -d                 # 임시 디렉토리 생성
```

## /proc — 프로세스 정보 (가상 파일시스템)

```
실제 파일이 아님
커널이 메모리에서 동적으로 생성
프로세스 / 시스템 정보를 파일로 표현
```

```bash
cat /proc/cpuinfo        # CPU 정보
cat /proc/meminfo        # 메모리 정보
cat /proc/1/status       # PID 1 프로세스 상태
ls /proc/                # 실행 중인 프로세스 PID 목록
```

## /dev — 장치 파일

```
하드웨어 장치를 파일로 표현
"모든 것이 파일" 의 핵심 개념
```

```bash
/dev/sda             # 첫 번째 하드디스크
/dev/sda1            # 첫 번째 파티션
/dev/null            # 블랙홀 (입력 버리기)
/dev/zero            # 무한 0 바이트 스트림
/dev/random          # 랜덤 데이터
/dev/tty             # 현재 터미널
```

---

---

# ③ 절대경로 vs 상대경로 ⭐️

## 절대경로 — / 로 시작

```
루트(/) 에서 시작하는 전체 경로
어디에 있든 항상 같은 위치를 가리킴
```

```bash
/etc/nginx/nginx.conf       # 항상 이 파일
/home/gong/projects/app.py  # 항상 이 파일
/var/log/syslog             # 항상 이 파일
```

## 상대경로 — 현재 위치 기준

```
현재 디렉토리(.) 기준으로 이동
현재 위치가 바뀌면 가리키는 파일도 바뀜
```

```bash
# 현재 위치: /home/gong/

projects/app.py      # /home/gong/projects/app.py
./app.py             # /home/gong/app.py  (. = 현재)
../other/file.txt    # /home/other/file.txt  (.. = 상위)
../../etc/hosts      # /etc/hosts
```

## 특수 경로 기호

```
.    현재 디렉토리
..   부모 디렉토리 (한 단계 위)
~    홈 디렉토리 (/home/사용자명)
-    이전 디렉토리 (cd - 에서 사용)
```

## 비교 예시

```bash
# 현재 위치: /home/gong/projects/

# 절대경로로 이동
cd /etc                         # 어디서든 동일
cd /var/log

# 상대경로로 이동
cd ..                           # /home/gong/
cd ../..                        # /home/
cd ../../etc                    # /etc

# 절대 vs 상대 언제 쓰나
# 스크립트 → 절대경로 (현재 위치에 의존 안 하도록)
# 터미널 → 상대경로 (짧게 치려고)
```

---

---

# ④ 경로 관련 명령어

```bash
pwd                  # 현재 절대경로 출력
ls -la /etc          # 특정 경로 목록
find / -name "*.log" # 전체에서 파일 찾기
which python3        # 실행 파일 위치 (절대경로)
whereis nginx        # 바이너리 / 소스 / 매뉴얼 위치
realpath ./file.txt  # 상대경로 → 절대경로 변환
```

---

---

# ⑤ 데이터 엔지니어 실무 경로

```bash
# Docker 볼륨 마운트할 때 절대경로 필수
volumes:
  - /var/lib/postgresql/data:/var/lib/postgresql/data
  - ./dags:/opt/airflow/dags     # ./ 상대경로 가능

# Airflow DAG 경로
/opt/airflow/dags/               # Airflow 기본 DAG 위치

# 로그 확인
/var/log/syslog                  # 시스템 로그
tail -f /var/log/nginx/error.log # Nginx 에러 실시간

# 설정 파일 편집
sudo vim /etc/hosts
sudo vim /etc/nginx/nginx.conf
```

---

---

# 한눈에 정리

|디렉토리|역할|예시|
|---|---|---|
|`/etc`|설정 파일|`/etc/passwd` / `/etc/nginx/`|
|`/var`|가변 데이터|`/var/log/` / `/var/lib/postgresql/`|
|`/home`|사용자 홈|`/home/gong/`|
|`/root`|root 홈|`/root/`|
|`/usr`|설치된 프로그램|`/usr/bin/python3`|
|`/usr/local`|직접 설치|`/usr/local/bin/`|
|`/tmp`|임시 파일|재부팅 시 삭제|
|`/proc`|프로세스 정보|`/proc/cpuinfo`|
|`/dev`|장치 파일|`/dev/null` / `/dev/sda`|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|스크립트에서 상대경로 써서 실행 위치마다 다름|현재 디렉토리 의존|스크립트 안에서 절대경로 사용|
|`/var/log` 가득 차서 서비스 장애|로그 정리 안 함|logrotate / 주기적 삭제|
|`/tmp` 파일 믿고 썼는데 재부팅 후 사라짐|임시 파일 특성|영구 저장은 `/var` 또는 홈 사용|
|`~` 가 root 셸에서 `/home/user` 아님|root 홈은 `/root`|root 에서 `~` = `/root`|