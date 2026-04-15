---
aliases:
  - sudo
  - 관리자 권한
  - root
  - 슈퍼유저
  - su
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_User_Group]]"
  - "[[Linux_File_Types]]"
  - "[[Linux_OpenSSL]]"
  - "[[Linux_Shell_Script]]"
---
# Superuser — sudo · su · root

## 한 줄 요약

```
일반 사용자가 잠시 '최고 관리자(Root)' 의 가면을 쓰고
강력한 명령을 내리게 해주는 마법의 명령어

SuperUser DO = "슈퍼유저가 하란다!"
```

---

---

# 왜 필요한가?

```
리눅스는 보안을 위해 평소에는 힘없는 일반 유저(ubuntu) 로 로그인
→ 실수로 중요 시스템 파일을 지우는 대참사를 막기 위해

하지만 프로그램 설치, 설정 변경 시에는 강력한 힘이 필요
→ 로그아웃 없이 명령어 앞에 sudo 만 붙여서 임시 관리자 승인
```

**실무에서 "Permission denied" 를 만났을 때 90%는 sudo 가 필요한 상황이다.**

```bash
sudo apt-get install python3   # 프로그램 설치
sudo vi /etc/hosts             # 시스템 파일 수정
sudo docker ps                 # Docker 권한 문제
```

---

---

# ① Root vs Sudo

```
Root (절대 권력자)
  윈도우의 Administrator 와 같음
  시스템의 모든 것을 파괴할 수 있는 신(God)
  프롬프트가 $ 대신 # 으로 표시됨
  ubuntu@server:~$   ← 일반 유저
  root@server:~#     ← root

Sudo (대리인)
  일반 유저가 잠시 Root 권한을 빌려 쓰는 것
  비밀번호를 물어보면 내 계정(ubuntu) 비밀번호 입력
  화면에 글자가 안 찍혀도 입력되고 있으니 당황 금지!
```

---

---

# ② 핵심 명령어

## sudo — 한 번만 권한 빌리기

```bash
# 일반 명령어 앞에 sudo 붙이기
sudo apt-get update
sudo systemctl restart nginx
```

## sudo !! — 방금 실패한 명령어 재실행 ⭐️

```
방금 입력한 명령어가 "권한 거부" 당했을 때
다시 타이핑할 필요 없이 sudo !! 한 방으로 해결
```

```bash
apt-get update
# [에러] Permission denied... (아 맞다 sudo!)

sudo !!
# → 자동으로 'sudo apt-get update' 실행
```

## sudo -i — 아예 root 로 변신

```
명령어마다 sudo 치기 귀찮을 때
아예 root 계정으로 로그인해버리는 것
작업 끝나면 반드시 exit 로 일반 유저로 돌아올 것
```

```bash
sudo -i
# 프롬프트: root@server:~#

# root 로 여러 작업 처리
apt-get update
vi /etc/hosts

# 일반 유저로 복귀
exit
```

## su — 다른 유저로 전환

```bash
# root 계정으로 전환 (root 비밀번호 필요)
su root
su -   # - 옵션: root 의 환경변수까지 로드

# 특정 유저로 전환
su - username

# 돌아오기
exit
```

---

---

# ③ 시스템 파일 수정 — sudo nano / sudo vi

```
/etc/ 아래 파일들은 시스템 설정 파일
→ 일반 유저는 읽기만 가능, 수정 불가
→ sudo 없이 열면 저장 시 "Permission denied" 또는 저장 자체가 안 됨
```

## /etc/hosts — 도메인-IP 매핑 파일

```
/etc/hosts 는 DNS 보다 먼저 조회되는 로컬 도메인 설정 파일
"이 이름은 이 IP 야" 를 직접 등록해두는 곳

실무 활용:
  Docker 컨테이너 이름으로 통신할 때
  kafka 라는 이름으로 127.0.0.1 을 가리키게 설정
  → 맥북에서 kafka:9092 로 접속 가능하게 만들기
```

```bash
# /etc/hosts 열기 (맥북은 관리자 비밀번호 요구)
sudo nano /etc/hosts

# 또는 vi 사용
sudo vi /etc/hosts
```

```
파일 내용 예시:
  127.0.0.1   localhost
  127.0.0.1   kafka        ← 추가: kafka 이름 → 내 컴퓨터로 해석
  127.0.0.1   postgres     ← 추가: postgres 이름 → 내 컴퓨터로 해석
```

```
sudo nano /etc/hosts 분해:
  sudo        관리자 권한으로
  nano        텍스트 에디터로
  /etc/hosts  이 파일을 열어라

  → sudo 없이 열면 읽기 전용 → 저장 불가
  → sudo 붙여야 수정 + 저장 가능
```

## nano 저장 단축키

```
Ctrl + O   → 저장 (Write Out)
Enter      → 파일명 확인
Ctrl + X   → 종료

vi 사용 시:
  i         → 입력 모드
  ESC       → 명령 모드
  :wq       → 저장 후 종료
  :q!       → 저장 없이 강제 종료
```

---

---

# ⑤ 현재 사용자 확인 — whoami / id

```bash
whoami         # 현재 사용자명 출력
# labex

id             # UID / GID / 소속 그룹 전부
# uid=1000(labex) gid=1000(labex) groups=1000(labex),27(sudo)

id labex       # 특정 사용자 정보
```

```
whoami 활용:
  sudo 명령 후 현재 내가 누구인지 확인
  스크립트에서 현재 유저 기반 처리
  sudo -i 로 root 가 됐는지 확인
```

```bash
# sudo -i 전후 비교
whoami         # labex
sudo -i
whoami         # root  ← root 로 전환됨
exit
whoami         # labex  ← 복귀
```

---

---

# ⑥ sudo 로 만든 파일 소유권 주의 ⭐️

```
sudo 로 명령 실행 → 생성된 파일 소유자 = root
나중에 수정 / 삭제 시 Permission denied
```

```bash
sudo tar -cvf ~/backup.tar /home
ls -l ~/backup.tar
# -rw-r--r-- 1 root root 10240 backup.tar
#              ↑ 소유자가 root!

# 소유권 내 계정으로 되돌리기
sudo chown $USER:$USER ~/backup.tar
ls -l ~/backup.tar
# -rw-r--r-- 1 labex labex 10240 backup.tar
```

```
원칙:
  홈 디렉토리(~) 에서는 sudo 없이 작업
  /etc / /var / /usr 등 시스템 영역만 sudo

  sudo 로 만든 파일 내가 다루려면:
  → sudo chown $USER:$USER 파일명
```

```
누가 sudo 를 쓸 수 있는지 정의하는 파일
직접 vi 로 열면 안 됨 → visudo 명령어로만 수정
(문법 오류 시 sudo 전체 잠김 방지)
```

```bash
# sudoers 파일 수정 (반드시 visudo 사용)
sudo visudo

# 특정 유저에게 sudo 권한 부여
username ALL=(ALL:ALL) ALL

# 비밀번호 없이 sudo 허용 (자동화 스크립트 등)
username ALL=(ALL) NOPASSWD:ALL
```

---

---

# 초보자가 자주 하는 실수

## ① 비밀번호를 쳤는데 화면에 안 나와요

```
리눅스 보안 정책상 비밀번호 입력은 화면에 아무것도 표시 안 됨
별표 * 도 안 나옴 — 고장난 게 아님
침착하게 입력하고 엔터
```

## ② 홈 디렉토리에서 sudo 쓰면 안 됨

```
내 홈 디렉토리(~) 에서 sudo 로 파일 만들면
파일 소유자가 root 가 되어버림
→ 나중에 내가 수정/삭제 못 함
→ chown 으로 소유권 되돌려야 하는 악순환

원칙:
  내 집(~, /home/ubuntu/) 에서는 sudo 없이
  공공장소(/etc/, /var/, /usr/) 를 건드릴 때만 sudo
```

```bash
# ❌ 홈에서 sudo 로 파일 생성
sudo touch ~/myfile.txt   # 소유자가 root 가 됨

# ✅ 홈에서는 그냥
touch ~/myfile.txt        # 소유자가 내 계정

# 소유권 확인
ls -la ~/myfile.txt
```

## ③ sudo -i 후 exit 안 하고 방치

```
sudo -i 로 root 가 된 상태에서 그냥 나가면
다음 사람도 root 권한으로 작업 가능
→ 반드시 exit 로 일반 유저로 복귀
```

|실수|해결|
|---|---|
|`Permission denied` 에러|명령어 앞에 `sudo` 추가|
|sudo 쳤는데 비밀번호 화면 먹통|그냥 입력 중 — 엔터 누르면 됨|
|방금 실패한 명령어 다시 실행|`sudo !!`|
|홈에서 만든 파일 수정 못 함|`sudo chown $USER:$USER 파일명`|
|`/etc/sudoers` 수정 후 sudo 잠김|`visudo` 로만 수정할 것|