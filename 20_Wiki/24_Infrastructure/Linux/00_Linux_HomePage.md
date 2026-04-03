
>목표: 리눅스 개념을 제대로 이해하고 데이터 엔지니어 실무 + 리눅스 마스터 1급 동시에 준비

---

---

## Level 0. 리눅스란 무엇인가

```
리눅스를 쓰기 전에 이해해야 하는 핵심 개념들
왜 리눅스가 서버 OS 의 표준인지, 구조가 어떻게 생겼는지
```

> 운영체제 이론 기초 → [[CS_Operating_System]] 먼저 읽기 권장 프로세스 / 메모리 / 파일시스템 / 스케줄링 개념이 여기서 나옴

|노트|핵심 개념|
|---|---|
|[[CS_Operating_System]]|OS 역할 / 프로세스·스레드 / 스케줄링 / 메모리 / inode / 데드락|
|[[Linux_Overview]]|커널 / 쉘 / 배포판 / Windows vs Linux|
|[[Linux_Filesystem]]|디렉토리 트리 / /etc /var /home /usr 역할 / 절대경로 vs 상대경로|
|[[Linux_Basic_Commands]]|ls / cd / pwd / mkdir / rm / cp / mv / man|

---

---

## Level 1. 파일과 텍스트 처리

```
리눅스는 "모든 것이 파일"
파일을 읽고 / 찾고 / 변환하는 능력이 핵심
로그 분석, 데이터 전처리에 매일 사용
```

| 노트                      | 핵심 개념                                             |
| ----------------------- | ------------------------------------------------- |
| [[Linux_File_Commands]] | cat / head / tail / less / wc / sort / uniq / cut |
| [[Linux_Vim_Nano]]      | vim / nano / 모드 전환 / 저장·종료 / 이동·검색 / 상황별 선택       |
| [[Linux_Redirection]]   | 표준 입출력 / > >> < / pipe \| / tee / 2>&1            |
| [[Linux_Grep]]          | grep / -i -n -v -r -E / 정규식 / 파이프 조합              |
| [[Linux_Stream_Editor]] | sed — 치환·삭제·삽입 / 정규표현식 / 캡처그룹                     |
| [[Linux_Awk]]           | awk — 필드 분리 / 조건 처리 / 집계                          |
| [[Linux_Find]]          | find — 조건 검색 / -exec / xargs                      |
| [[Linux_Archive]]       | tar / gzip / bzip2 / zip — 압축 원리와 명령어             |

---

---

## Level 2. 프로세스와 시스템 자원

```
프로세스가 어떻게 동작하는지 이해
CPU / 메모리 / 디스크 자원을 모니터링하고 제어
Kafka / Spark 컨테이너 운영의 기초
```

|노트|핵심 개념|
|---|---|
|[[Linux_Process]]|프로세스 개념 / ps / top / htop / kill / signal|
|[[Linux_Background_Jobs]]|포그라운드 vs 백그라운드 / & / fg / bg / jobs / nohup|
|[[Linux_Systemctl]]|데몬(daemon) 개념 / systemctl / service / 부팅 자동 시작|
|[[Linux_Disk]]|파일시스템 마운트 / df / du / lsblk / fdisk / fsck|
|[[Linux_Memory]]|가상 메모리 / free / vmstat / /proc/meminfo / swap|
|[[Linux_Log]]|로그 시스템 구조 / journalctl / syslog / dmesg / logrotate|
|[[Linux_Performance]]|성능 분석 / sar / iostat / strace / ltrace|

---

---

## Level 3. 권한과 보안

```
리눅스 보안 모델의 핵심
"누가, 무엇을, 어떻게 할 수 있는가"
리눅스 마스터 1급 비중 가장 높은 영역
```

|노트|핵심 개념|
|---|---|
|[[File_Permissions]]|권한 개념 (rwx) / chmod / chown / chgrp / umask / SUID / SGID / Sticky bit|
|[[Superuser]]|root 개념 / sudo / su / sudoers / /etc/hosts|
|[[Linux_User_Group]]|사용자·그룹 관리 / useradd / usermod / groupadd / passwd / /etc/passwd /etc/shadow|
|[[Linux_SSH]]|공개키 암호화 개념 / ssh / scp / ssh-keygen / authorized_keys / config|
|[[Linux_OpenSSL]]|암호화 개념 / rand / dgst / enc / 인증서 / AES / RSA|
|[[Linux_Firewall]]|패킷 필터링 개념 / iptables / ufw / firewalld / 체인|
|[[Linux_PAM]]|플러그인 인증 구조 / /etc/pam.d / 모듈 유형|
|[[Linux_SELinux]]|강제 접근 통제 개념 / SELinux / AppArmor / 컨텍스트|

---

---

## Level 4. 쉘 스크립트와 자동화

```
반복 작업을 코드로 만들기
Airflow BashOperator / crontab 스케줄링의 기초
```

| 노트                              | 핵심 개념                                                      |
| ------------------------------- | ---------------------------------------------------------- |
| [[Linux_Shell_Script]]          | read/변수 / if / case / for / while / 함수 / 종료 코드             |
| [[Linux_Shell_Arrays]]          | 배열, readArray, while read                                  |
| [[Linux_Shell_Arithmetic]]      | $(()) / bc / bc -l / scale / printf, 반올림                   |
| [[Linux_Environment_Variables]] | 환경변수 개념 / export / .bashrc / .bash_profile / source / PATH |
| [[Linux_Scheduling_Crontab]]    | cron 데몬 / crontab 문법 / 실전 패턴 / 로그 리다이렉션                    |
| [[Linux_String_Processing]]     | 변수 치환 / 슬라이싱 / 대소문자 변환 / 정규표현식 연동                          |

---

---

## Level 5. 네트워크

```
네트워크 동작 원리부터 서버 설정까지
Docker 네트워크 트러블슈팅 / 서비스 포트 확인에 매일 사용
리눅스 마스터 1급 출제 비중 30%
```

>CS Basics [[CS_HTTP_Basics]] , [[CS_TCP_IP]], [[CS_REST_API_Methods]] 참고 

| 노트                         | 핵심 개념                                                                    |
| -------------------------- | ------------------------------------------------------------------------ |
| [[Linux_Network_Basic]]    | OSI 7계층 / IP / 서브넷 / 게이트웨이 / DNS / TCP vs UDP                            |
| [[Linux_Network_Commands]] | ifconfig / ip addr / ping / traceroute / netstat / ss / nmap             |
| [[Linux_Port]]             | 포트 개념 / lsof / netstat -tulnp / curl / wget                              |
| [[Linux_Network_Config]]   | /etc/hosts / /etc/resolv.conf / /etc/network/interfaces / NetworkManager |
| [[Linux_NFS_Samba]]        | 파일 공유 개념 / NFS 마운트 / Samba 설정                                            |
| [[Linux_DNS_DHCP]]         | DNS 동작 원리 / BIND 설정 / DHCP 임대 개념                                         |
| [[Linux_Apache_Nginx]]     | 웹서버 구조 / Virtual Host / 설정 파일 / 로그                                       |

---

---

## Level 6. 패키지 관리

```
소프트웨어를 어떻게 설치하고 관리하는지
패키지 시스템의 동작 원리 이해
```

| 노트                        | 핵심 개념                                               |
| ------------------------- | --------------------------------------------------- |
| [[Linux_Package_APT]]     | 패키지 관리자 개념 / apt / apt-get / dpkg / 저장소(repository) |
| [[Linux_Package_YUM]]     | rpm 구조 / yum / dnf / rpm 명령어 / spec 파일              |
| [[Linux_Compile_Install]] | 소스 컴파일 / ./configure / make / make install / 의존성    |
| [[Linux_Python_Env]]      | pip / venv / pyenv / requirements.txt               |

---

---

## Level 7. 고급 시스템 관리 (1급 심화)

```
리눅스 내부 동작 원리를 깊이 이해
커널부터 부팅, 스토리지 고급 기술까지
리눅스 마스터 1급 시스템 프로그래밍 영역 20%
```

|노트|핵심 개념|
|---|---|
|[[Linux_Boot_Process]]|BIOS / UEFI → GRUB → 커널 로딩 → init → systemd 전체 부팅 흐름|
|[[Linux_Runlevel]]|Runlevel 0~6 개념 / systemd target / 기본 타겟 변경|
|[[Linux_Kernel]]|커널 역할 / 모듈 구조 / lsmod / modprobe / insmod / 커널 컴파일|
|[[Linux_LVM]]|물리볼륨 / 볼륨그룹 / 논리볼륨 개념 / pvcreate / vgcreate / lvcreate|
|[[Linux_RAID]]|RAID 개념 / 0(스트라이핑) / 1(미러링) / 5 / 6 / mdadm|
|[[Linux_Backup]]|백업 전략 / rsync / dump / restore / Amanda|

---

---
