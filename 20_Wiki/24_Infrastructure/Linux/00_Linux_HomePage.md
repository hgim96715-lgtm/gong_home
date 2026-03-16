[[01_Linux_Roadmap]] : 흐름도

> Linux 연습 [리눅스 연습](https://labex.io/ko/labs/linux-your-first-linux-lab-270253?course=quick-start-with-linux) 참고
> [[흐름도 설명]]

## Level 0. 리눅스의 세계관 (Architecture & History)

> "명령어를 치기 전에, 내가 어디에 들어와 있는지 지도를 먼저 봐요."

- [[Linux_History_and_Origin]] : "리눅스는 왜 태어났고, 왜 무료인가?" (`GNU`, `Linus Torvalds`, `오픈소스 철학`)
- [[Open_Source_License_GPL]] : "공짜로 쓰되, 왜 우리는 오픈소스 생태계에 기여해야 하는가?" (`GPL`, `MIT`, `Apache`, `카피레프트`)
- [[Linux_vs_Unix]] : "리눅스와 맥(Unix 기반)은 왜 비슷하면서도 다른가?" (`POSIX`, `macOS`, `족보`, `호환성`)
- [[Linux_Architecture]] : "리눅스 내부의 심장(Kernel)과 입(Shell)은 어떻게 대화하는가?" (`Kernel`, `Shell`, `System Call`, `User Space`)
- [[Linux_Distributions]] : "왜 서버마다 `apt`를 쓰기도 하고 `yum`을 쓰기도 하는가?" (`Ubuntu`, `CentOS`, `Debian`, `apt`, `yum`, `dnf`)
- [[Linux_Fundamental_Rules]] : "대소문자, 공백, 루트 권한... 리눅스에서 절대 어기면 안 되는 법도는 무엇인가?" (`whoami`, `id`, `대소문자 구분`, `공백 주의`)
- [[Filesystem_Hierarchy_Standard]] : 리눅스 계층 구조, 디렉토리 구조 (`/etc`, `/var`, `/home`, `/usr`, `/tmp`, `지도 보기`)
- [[Linux_Docker_Setup]] : 실습 환경 구축 (`Docker`, `컨테이너`, `이미지`, `나만의 연습장`)

---

## Level 1. 터미널 생존 신고 (Basics)

> "검은 화면이 더 이상 무섭지 않아요."

- [[Shell_Interface]] : 쉘(Shell)이란 무엇인가? (`GUI vs CLI`, `bash`, `zsh`, `프롬프트`)
- [[File_Navigation]] : 길 잃지 않고 이동하기 (`pwd`, `ls`, `cd`, `절대경로`, `상대경로`, `~`)
- [[Linux_File_Types]] : 리눅스의 7가지 파일 타입 (`-일반`, `d디렉토리`, `l링크`, `s소켓`, `b블록`)
- [[Links_and_Inodes]] : 링크와 아이노드 완벽 이해 (`ln`, `stat`, `하드링크`, `심볼릭링크`, `inode번호`)
- [[File_Management]] : 파일 생성과 삭제, 복사 (`mkdir`, `touch`, `cp`, `mv`, `rm`, `echo`, `-r 재귀`)
- [[Shell_Wildcards]] : 파일 찾기의 조커 카드 (`*`, `?`, `[]`, `글로브 패턴`)
- [[File_Compression]] : 파일 압축 및 해제 (`tar`, `gzip`, `-cvf`, `-xvf`, `.tar.gz`)
- [[File_Content_Viewing]] : 파일 내용 훑어보기 (`cat`, `head`, `tail`, `less`, `seq`, `-n 줄수`)
- [[Redirection_Pipe]] : 명령어끼리 연결하기 (`|`, `>`, `>>`, `&&`, `||`, `;`, `stdin/stdout/stderr`)
- [[Terminal_Editors]] : 파일 내용 깊게 수정하기 (`vi`, `nano`, `i삽입`, `ESC`, `:wq`, `:q!`)
- [[Help_Commands]] : 모르는 명령어 검색하기 (`man`, `--help`, `tldr`)

---

## Level 2. 환경 설정과 변수 (Environment)

> "내 컴퓨터가 나를 알아보게 만들어요."

- [[Environment_Variables]] : 환경 변수의 개념과 설정 (`env`, `export`, `$변수명`, `printenv`)
- [[Shell_Parameter_Expansion]] : 변수 가공의 마법 (`${var,,}`, `${var:-default}`, `${var#prefix}`, `${#var}`)
- [[Path_Variable]] : 마법의 변수 PATH 이해하기 (`$PATH`, `which`, `명령어 탐색 순서`)
- [[Shell_Config]] : 설정을 영구 저장하고 적용하기 (`.zshrc`, `.bashrc`, `source`, `alias`)

---

## Level 3. 권한과 보안 (Permissions)

> "내 파일은 내가 지킨다! 허락받고 들어와."

- [[File_Permissions]] : 파일의 권한과 소유권 완벽 정리 (`chmod`, `chown`, `rwx`, `755`, `644`, `8진수`)
- [[Superuser]] : 관리자 권한의 힘과 책임 (`sudo`, `su`, `root`, `/etc/sudoers`)
- [[Linux_OpenSSL]] : 키 생성 · 인증서 · 암호화 도구 (`openssl rand`, `base64`, `SECRET_KEY 생성`, `SHA256`, `인증서 확인`)

---

## Level 4. 프로세스와 네트워크 (Engine)

> "서버가 죽었는지 살았는지 확인해요."

- [[SSH_Connection]] : 안전한 원격 접속과 파일 전송 (`ssh`, `scp`, `키페어`, `-i`, `공개키/개인키`)
- [[Process_Management]] : 실행 중인 프로그램 관리 (`ps`, `aux`, `top`, `kill`, `daemon`, `PID`)
- **[[System_Uptime]]** : 서버 부하량 확인 (`uptime`, `load average`, `1분/5분/15분`)
- [[Disk_Management]] : 디스크 용량 확인 및 청소 (`df`, `du`, `-h`, `-sh`, `용량 단위`)
- [[Linux_Signals]] : 프로세스 종료와 신호 (`kill -9`, `SIGTERM`, `SIGKILL`, `SIGHUP`)
- [[Background_Jobs]] : 백그라운드 실행 (`&`, `nohup`, `jobs`, `fg`, `bg`, `disown`)
- [[File_Transfer]] : 파일 다운로드 및 API 요청 (`curl`, `wget`, `-O`, `-X POST`, `헤더`)
- [[Network_Troubleshooting]] : 연결 상태 및 경로 진단 (`ping`, `nslookup`, `traceroute`, `dig`, `DNS`)
- [[Network_Status]] : 포트 및 연결 확인 (`netstat`, `lsof`, `ss`, `-tulpn`, `LISTEN`)
- [[Time_Synchronization]] : 서버 시간 동기화 (`ntp`, `chronyd`, `timedatectl`, `UTC`)
- [[Package_Management]] : 프로그램 설치와 관리 (`apt`, `apt-get`, `install`, `update`, `upgrade`)

---

## Level 5. 데이터 엔지니어의 무기 (Text Processing)

> "수만 줄의 로그에서 원하는 정보만 딱! 골라내요."

- [[Linux_Filtering_Text]] : 원하는 글자 찾기 (`grep`, `-i`, `-r`, `-n`, `-v`, `정규식`)
- [[Linux_Text_Transformation_tr]] : 문자 치환 및 삭제의 기본 (`tr`, `대소문자 변환`, `특수문자 삭제`)
- [[Linux_Text_Slicing_cut]] : 칼로 자르듯 원하는 열만 쏙! (`cut`, `-d 구분자`, `-f 필드번호`, `CSV 처리`)
- [[Linux_Stream_Editor]] : 데이터 가공의 기초 (`awk`, `sed`, `$1 $2 필드`, `s/찾기/바꾸기/`, `NR`)
- [[Linux_JSON_Processing]] : JSON 데이터 파싱의 신 (`jq`, `.key`, `[]`, `파이프`, `API 응답 처리`)
- [[Linux_Data_Statistics]] : 데이터 개수 세기 및 정렬 (`wc`, `sort`, `uniq`, `-l 줄수`, `-c 빈도`)
- [[Linux_File_Comparison]] : 파일 내용 비교 및 검증 (`diff`, `cmp`, `patch`, `변경점 추적`)

---

## Level 6. 자동화 스크립트 (Automation)

> "반복 작업은 컴퓨터에게 시키고 퇴근해요."

- [[Linux_Shell_Script]] : 쉘 스크립트 기초 문법 (`if`, `for`, `while`, `var`, `#!/bin/bash`, `실행권한`)
- [[Shell_Input_Output]] : 사용자와 대화하기 (`read`, `$1 위치인자`, `stdin`, `echo`, `-p 프롬프트`)
- [[Shell_Arithmetic]] : 쉘에서의 산술 연산 (`$(())`, `expr`, `bc`, `소수점 계산`)
- [[Linux_Scheduling_Crontab]] : 정해진 시간에 실행하기 (`crontab`, `-e`, `* * * * *`, `분/시/일/월/요일`)
- [[Linux_Shell_Arrays]] : 배열로 데이터 묶어서 처리하기 (`배열 선언`, `인덱스 접근`, `@/*`, `for 루프`, `배열 길이`, `슬라이싱`)