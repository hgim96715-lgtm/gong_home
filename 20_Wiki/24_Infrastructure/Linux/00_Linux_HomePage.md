

## Level 0. 리눅스의 세계관 (Architecture & History)

> "명령어를 치기 전에, 내가 어디에 들어와 있는지 지도를 먼저 봐요."

- [[Linux_History_and_Origin]]: "리눅스는 왜 태어났고, 왜 무료인가?" (철학적 배경)
- [[Open_Source_License_GPL]]: "공짜로 쓰되, 왜 우리는 오픈소스 생태계에 기여해야 하는가?" (공유의 정신)
- [[Linux_vs_Unix]]: "리눅스와 맥(Unix 기반)은 왜 비슷하면서도 다른가?" (족보 이해)
- [[Linux_Architecture]]: "리눅스 내부의 심장(Kernel)과 입(Shell)은 어떻게 대화하는가?" (시스템 구조)
- [[Linux_Distributions]]: "왜 서버마다 `apt`를 쓰기도 하고 `yum`을 쓰기도 하는가?" (가문별 명령어 차이)
- [[Linux_Fundamental_Rules]]: "대소문자, 공백, 루트 권한... 리눅스에서 절대 어기면 안 되는 법도는 무엇인가?" (실전 매너)
- [[Filesystem Hierarchy Standard]] : 리눅스 계층 구조, 디렉토리 구조 (지도 보기)
- [[Linux_Docker_Setup]] : 실습 환경 구축 (Docker로 나만의 연습장 만들기)

## Level 1. 터미널 생존 신고 (Basics)

> "검은 화면이 더 이상 무섭지 않아요."

- [[Shell_Interface]] : 쉘(Shell)이란 무엇인가? (GUI vs CLI)
- [[File_Navigation]] : 길 잃지 않고 이동하기 (`pwd`, `ls`, `cd`)
- [[File_Management]] : 파일 생성과 삭제, 복사 (`mkdir`, `touch`, `cp`, `mv`, `rm`)
- [[File_Content_Viewing]] : 파일 내용 훑어보기 (`cat`, `head`, `tail`, `less`, + `seq`)
- [[Redirection_Pipe]] : 명령어끼리 연결하기 (`|`, `>`, `>>`)    
- [[Terminal_Editors]] : 파일 내용 깊게 수정하기 (`vi`, `nano`) 
- [[Help_Commands]] : 모르는 명령어 검색하기 (`man`, `--help`)

## Level 2. 환경 설정과 변수 (Environment)

> "내 컴퓨터가 나를 알아보게 만들어요."

- [[Environment_Variables]] : 환경 변수의 개념과 설정 (`env`, `export`)  다시 
- [[Path_Variable]] : 마법의 변수 PATH 이해하기 (`$PATH`) 다시 
- [[Shell_Config]] : 설정을 영구 저장하고 적용하기 (`.zshrc`, `.bashrc`, `source`)

## Level 3. 권한과 보안 (Permissions)

> "내 파일은 내가 지킨다! 허락받고 들어와."

- [[File_Permissions]] : `rwx`와 숫자 표기법 (`chmod`)
- [[Ownership]] : 주인과 그룹 바꾸기 (`chown`)
- [[Superuser]] : 관리자 권한의 힘과 책임 (`sudo`)

## Level 4. 프로세스와 네트워크 (Engine)

> "서버가 죽었는지 살았는지 확인해요."

- [[Process_Management]] : 실행 중인 프로그램 관리 (`ps`, `kill`, `top`)
- [[Background_Jobs]] : 백그라운드 실행 (`&`, `nohup`)
- [[Network_Basics]] : 포트 확인과 연결 테스트 (`lsof`, `netstat`, `curl`)

## Level 5. 데이터 엔지니어의 무기 (Text Processing)

> "수만 줄의 로그에서 원하는 정보만 딱! 골라내요."

- [[Filtering_Text]] : 원하는 글자 찾기 (`grep`)
- [[Stream_Editor]] : 데이터 가공의 기초 (`awk`, `sed`)
- [[Data_Statistics]] : 데이터 개수 세기 및 정렬 (`wc`, `sort`, `uniq`)

## Level 6. 자동화 스크립트 (Automation)

> "반복 작업은 컴퓨터에게 시키고 퇴근해요."

- [[Shell_Scripting]] : 쉘 스크립트 기초 (`.sh` 파일 만들기)
- [[Linux_Scheduling_Crontab]] : 정해진 시간에 실행하기 (`crontab`) 다시 
