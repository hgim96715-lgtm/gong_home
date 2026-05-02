
```
데이터 엔지니어가 서버를 다루는 언어
GUI 없이 명령어 하나로 시스템 전체를 제어하는 기술
```

---

---

## Level 0. 개념 잡기

```
Linux가 뭔지, 왜 데이터 엔지니어에게 필수인지
```

| 노트                            | 핵심 개념                                                               |
| ----------------------------- | ------------------------------------------------------------------- |
| [[Linux_Concept_Overview]]    | 커널 / 쉘 / 배포판(Ubuntu) / CLI vs GUI / 왜 서버는 Linux인가                   |
| [[Linux_Directory_Structure]] | / (루트) / home / etc / var / tmp / opt / 절대경로 vs 상대경로                |
| [[Linux_Permission_Model]]    | rwx / 소유자·그룹·기타 / chmod / chown / uid / gid / sudo                  |
| [[Linux_User_Group]]          | useradd / userdel / usermod / passwd / groupadd / 계정 잠금 / /etc/skel |

---

---

## Level 1. 파일 시스템 조작

```
서버에서 파일과 폴더를 자유자재로 다루자
```

|노트|핵심 개념|
|---|---|
|[[Linux_Directory_Commands]] ⭐|mkdir / cd / ls -F / pwd / tree / 프로젝트 구조 설계|
|[[Linux_File_Move_Copy]] ⭐|mv (이동·이름변경) / cp / cp .bak 백업 패턴 / 롤백|
|[[Linux_File_Delete]]|rm / rm -rf / rmdir / 와일드카드(*) 패턴 삭제 / 주의사항|
|[[Linux_Archive_Compress]] ⭐|tar -czf / tar -tzf / tar -xzf / gzip / 로그 로테이션 패턴|

---

---

## Level 2. 시스템 모니터링

```
서버가 지금 어떤 상태인지 읽는 법
```

|노트|핵심 개념|
|---|---|
|[[Linux_System_Info]] ⭐|whoami / uname -a / uptime / id / hostname / 시스템 점검 루틴|
|[[Linux_Process_Monitor]] ⭐|top / htop / ps aux / kill / PID / Load Average / 좀비 프로세스|
|[[Linux_Disk_Memory]]|df -h / du -sh / free -h / 디스크 100% 장애 대응 / 용량 확보|
|[[Linux_Network_Check]]|ip addr / curl / wget / netstat / ss / 포트 확인 / ping|
|[[Linux_Log]] ⭐|dmesg / journalctl / syslog / logrotate / 에러 로그 추적|

---

---

## Level 3. 텍스트 처리 & 검색

```
로그 파일에서 필요한 정보를 뽑아내자
```

| 노트                        | 핵심 개념                                                   |
| ------------------------- | ------------------------------------------------------- |
| [[Linux_Text_Commands]]   | cat / less / head / tail -f / 실시간 로그 모니터링               |
| [[Linux_Search_Filter]] ⭐ | grep / grep -r / grep -i / -E / 파이프(\|) / 로그에서 ERROR 추출 |
| [[Linux_Text_Processing]] | awk / sed / cut / sort / uniq / wc / 데이터 전처리 기초         |
| [[Linux_Redirect]] ⭐      | > (덮어쓰기) / >> (추가) / < / 2> stderr / 보고서 자동 생성          |
| [[Linux_Diff]]            | diff / diff -r / 스테이징 vs 프로덕션 비교 / 누락 파일 찾기             |

---

---

## Level 4. 쉘 스크립트 & 자동화

```
반복 작업을 스크립트로 만들어서 자동화하자
```

|노트|핵심 개념|
|---|---|
|[[Shell_Script_Basics]]|#!/bin/bash / 변수 / if / for / while / 실행권한 chmod +x|
|[[Shell_Cron_Job]] ⭐|crontab -e / 크론 표현식 / 로그 압축 자동화 / 백업 스케줄링|
|[[Shell_Script_Patterns]]|로그 로테이션 스크립트 / 디렉토리 백업 / 오류 처리 (exit code)|

---

---

## Level 5. 실전 팁

|노트|핵심 개념|
|---|---|
|[[Linux_Permission_Handling]]|sudo 없이 작업하기 / docker group 추가 / 권한 에러 패턴|
|[[Linux_SSH_Basics]]|ssh / scp / 키 기반 인증 / ~/.ssh/config / 원격 서버 접속|
|[[Linux_Environment_Variables]]|export / .bashrc / .bash_profile / PATH 추가 / env 확인|

---

---

## 프로젝트 적용

|노트|설명|
|---|---|
|[[Linux_Airflow_Log_Management]]|Airflow 로그 디렉토리 정리 / tar 압축 / cron 자동화|
|[[Linux_Kafka_Server_Check]]|Kafka 브로커 프로세스 확인 / 포트 점검 / 디스크 모니터링|
|[[Linux_Docker_Permission]]|Docker 소켓 권한 / 컨테이너 내부 Linux 명령어 활용|