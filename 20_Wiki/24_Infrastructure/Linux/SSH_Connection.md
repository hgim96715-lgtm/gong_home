---
aliases:
  - SSH
  - Secure Shell
  - 원격접속
  - scp
  - sftp
tags:
  - Linux
related:
  - "[[Superuser]]"
  - "[[Linux_Fundamental_Rules]]"
  - "[[00_Linux_HomePage(기존)]]"
---
## 개념 한 줄 요약

**"내 컴퓨터(Client)에서 멀리 떨어진 서버(Server)에 안전하게 접속해서, 마치 그 컴퓨터 앞에 앉아있는 것처럼 명령을 내리게 해주는 암호화된 터널이다."**

---
## 왜 필요한가? (Why)

**문제점:**
- 예전에 쓰던 `telnet` 같은 방식은 데이터가 암호화되지 않아서, 해커가 중간에서 아이디/비번을 낚아챌 수 있었다.

**해결책:**
- **SSH (Secure Shell)** 는 강력한 인증과 암호화를 제공한다.
- 접속뿐만 아니라 **파일 전송(SCP)** 이나 **명령어 실행** 등 네트워크상의 모든 작업을 안전하게 처리할 수 있다.

---
## 준비물 (Prerequisites)

접속하려면 딱 3가지가 필요하다.

1.  **SSH Client:** 내 컴퓨터에 깔려 있어야 함. (맥북/리눅스는 기본 설치됨, 윈도우는 PuTTY 등을 씀)
2.  **Server 주소:** 접속할 곳의 IP 주소 (`192.168.x.x`) 혹은 도메인 (`myserver.com`).
3.  **계정 정보:** 아이디(`username`)와 열쇠(비밀번호 혹은 `SSH Key`).

---
##  실무 코드 핵심 (Code Core Points)

터미널에서 가장 많이 쓰는 3대장 명령어다.

### ① 서버 접속하기 (`ssh`)

포트(Port) 옵션인 `-p` (소문자)를 주의해야 한다.

```bash
# 기본 문법: ssh 아이디@서버주소
# 1. 기본 접속 (22번 포트)
ssh ubuntu@192.168.0.10

# 포트 번호가 다를 때 (예: 2222번)
# (보안상 22번을 막아두는 경우가 많음)
ssh -p 2222 ubuntu@localhost

# 접속 종료 
exit
```

### ② SSH 키 관리 (Passwordless Login)

비밀번호 없이 **'열쇠 파일(Key File)'** 로 접속하는 방법. 
보안도 더 강력하고 편하다.

```bash
# 1. 열쇠 생성 (Key Generation)
# -t rsa: 암호화 알고리즘 (타입) 
# -b 4096: 키의 길이 (비트 수). 기본값(2048)보다 더 안전하게 만들기 위해 설정
# -f: 키 파일이 저장될 위치와 이름
ssh-keygen -t rsa -b 4096 -f ~/.ssh/custom_rsa_key

# 2. 서버에 내 열쇠(공개키) 등록하기 (Key Copy)
# 방법 A: 전용 명령어 사용 (가장 쉬움)
ssh-copy-id -p 2222 -i ~/.ssh/custom_rsa_key.pub ubuntu@localhost

# 방법 B: 수동 등록 (ssh-copy-id 명령어가 없을 때)
# "내 공개키를 읽어서 -> 서버로 보낸 뒤 -> 서버의 승인된 키 목록에 추가해라"
cat ~/.ssh/custom_rsa_key.pub | ssh -p 2222 ubuntu@localhost "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# 3. 개인키(Private Key)를 들고 접속하기
# -i: Identity file (내 비밀 열쇠 위치 지정)
ssh -i ~/.ssh/custom_rsa_key -p 2222 ubuntu@localhost
```

### ③ 파일 전송 (SCP & SFTP)

**주의:** SSH 접속은 `-p`(소문자)를 쓰지만, 
파일 전송(SCP/SFTP)은 포트 옵션으로 **`-P` (대문자)** 를 쓴다. (가장 많이 틀리는 부분!)

```bash
# 1. SCP (Secure Copy): 파일 하나 휙 던지기
# 내 컴퓨터 파일 -> 서버로 복사
scp localfile.txt username@hostname:/remote/directory/

# 포트 지정 시 (-P 대문자 주의)
scp -P 2222 localfile.txt username@hostname:/remote/directory/

# 2. SFTP (Secure FTP): 폴더 탐색기처럼 쓰기
# 접속 후 ls, cd, put, get 명령어 사용 가능
sftp -P 2222 username@hostname
```

---
## 상세 분석 (Detailed Analysis)

**"누가(Who) SSH를 쓰는가?"**

1. **Network Administrator (관리자):** 웹 서버를 켜거나 설정을 바꿀 때 원격으로 제어하기 위해 사용한다.

2. **Data Engineer (우리):**
    
    - 로컬에서 짠 코드를 서버에 올릴 때 (`scp`, `sftp`)
    - 무거운 데이터 처리를 고성능 서버에 시킬 때 접속해서 사용한다.


---
## 초보자가 자주 하는 실수 (Misconceptions)

1. **"비밀번호가 안 쳐져요!"**
    - `sudo` 때와 마찬가지로, SSH 비밀번호 입력 시 보안상 **화면에 아무것도 안 뜬다.** 고장 난 게 아니니 믿고 치고 엔터를 눌러라.
        
2. **"맥북은 뭘 설치해야 하죠?"**
    - 윈도우 유저들이 `PuTTY`나 `Xshell`을 까는 걸 보고 따라 하려는데, **맥북은 터미널 자체가 최고의 SSH 클라이언트다.** 아무것도 설치할 필요 없다.
        
3. **"접속했는데 내 파일이 없어요!"**
    - SSH로 접속하면 **서버 컴퓨터의 하드디스크**로 들어간 것이다. 내 맥북 바탕화면에 있는 파일은 당연히 안 보인다. 파일을 쓰고 싶으면 `scp`로 업로드부터 해야 한다.

---
## 7. Bonus: 실전 압축 연습 (Bandit Wargame) 

SSH와 리눅스 명령어를 가장 재미있게 익히는 방법이다.
**OverTheWire**라는 곳에서 운영하는 무료 해킹 실습 게임인데, 단계별로 권한 문제를 풀어야 다음 레벨로 넘어갈 수 있다.

### ① 접속 정보 (Target Info)

* **서버 주소 (Host):** `bandit.labs.overthewire.org`
* **포트 (Port):** `2220` (기본 22번이 아님! `-p 2220` 필수)
* **첫 번째 ID:** `bandit0`
* **첫 번째 PW:** `bandit0`

### ② 게임 방식 (The Loop)

이 게임은 **무한 반복 루프**로 진행된다.

1.  **접속:** 현재 레벨(예: `bandit0`)로 SSH 접속.
2.  **탐색:** `ls`, `cat`, `find` 등을 총동원해서 **다음 레벨(예: `bandit1`)의 비밀번호**가 적힌 파일을 찾아낸다.
3.  **획득:** 찾은 비밀번호를 복사(`Cmd+C`)한다.
4.  **탈출:** `exit` 명령어로 로그아웃.
5.  **진화:** 방금 얻은 비밀번호로 **다음 레벨(`bandit1`)에 접속!** (반복)

### ③ 바로 시작해보기 (Start Command)

터미널에 붙여넣고 엔터! (비밀번호: `bandit0`)

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220

# 단계가 올라갈수록 탈출후 bandit0->bandit1이렇게 변경해줘야한다
```

---
## Bonus: 방구석 실습실 (Docker로 가짜 서버 만들기) 

남의 서버(Bandit)는 관리자 권한이 없어서 `ssh-copy-id` 같은 설정을 못 해본다.
내 맥북에 **Docker**를 이용해 나만의 리눅스 서버를 띄우고, **키 생성부터 접속까지 완벽하게 시뮬레이션** 해보자.

>포트 충돌 문제를 피하기 위해, 안정적인 **2223번 포트**로 정리

### ① 실습 환경 구축 (Setup)

맥북 터미널에서 아래 명령어로 **가상 서버(Container)** 를 실행한다.
* **이미지:** `rastasheep/ubuntu-sshd` (SSH가 설치된 우분투)
* **포트:** `2223` (내 컴퓨터의 2223번을 서버의 22번과 연결)
* **비밀번호:** `root`

```bash
docker run -d -p 2223:22 --name ssh-lab rastasheep/ubuntu-sshd
```

### ② 실습 시나리오 (Scenario)

**Step 1. 열쇠공 되기 (Key Gen)** 비밀번호 없이 접속하기 위해 나만의 '디지털 열쇠'를 만든다.

```bash
# -f 옵션으로 파일 이름을 지정한다. (안 하면 id_rsa로 만들어짐)
ssh-keygen -t rsa -f ~/.ssh/lab_key
```

**Step 2. 열쇠 등록하기 (Copy ID)** 만들어진 **공개키(.pub)** 를 서버에게 건네준다. ("이 열쇠 가진 사람 문 열어줘!")

```bash
# 주의 1: 포트(-p)는 2223 (소문자)
# 주의 2: 아이디는 'root' (이 Docker 이미지의 기본 계정)
# 비밀번호 'root' 입력 필요
ssh-copy-id -p 2223 -i ~/.ssh/lab_key.pub root@localhost
```

**Step 3. 무사통과 접속 (Login)** 이제 비밀번호를 안 물어봐야 성공이다.

```bash
# -i 옵션으로 "방금 만든 그 열쇠(lab_key) 쓸게"라고 알려줌
ssh -i ~/.ssh/lab_key -p 2223 root@localhost
```

**Step 4. 파일 던지기 (SCP)** 내 맥북의 파일을 서버로 전송한다.

```bash
# 1. 테스트 파일 생성
echo "Hello Docker" > test.txt

# 2. 전송 (주의: SCP는 포트 옵션이 대문자 -P)
scp -P 2223 -i ~/.ssh/lab_key test.txt root@localhost:/root/
```

### ③ 뒷정리 (Clean Up)

실습이 끝나면 가상 서버와 키 파일을 삭제해서 깔끔하게 정리한다.

```bash
# 컨테이너 삭제
docker rm -f ssh-lab

# 키 파일 삭제
rm ~/.ssh/lab_key ~/.ssh/lab_key.pub
```

