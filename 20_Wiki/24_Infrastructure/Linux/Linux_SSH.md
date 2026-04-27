---
aliases:
  - SSH
  - ssh-keygen
  - authorized_keys
  - 원격접속
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Network_Commands]]"
  - "[[Linux_Superuser]]"
  - "[[Linux_User_Group]]"
---

# Linux_SSH — 원격 접속 & 공개키 인증

## 한 줄 요약

```
SSH = Secure Shell
암호화된 채널로 원격 서버에 안전하게 접속하는 프로토콜
비밀번호 로그인 vs 공개키 인증 두 가지 방식
```

---

---

# ① SSH 개념

```
SSH 가 없으면:
  텔넷(Telnet) → 패킷이 평문으로 전송 → 스니핑으로 비밀번호 도청 가능

SSH 가 있으면:
  모든 통신 암호화 → 도청해도 해독 불가

포트: 22 (기본값)
서버: sshd 데몬 (openssh-server)
클라이언트: ssh 명령어
```

## 비밀번호 로그인 vs 공개키 인증

|구분|비밀번호|공개키|
|---|---|---|
|보안|보통|높음|
|자동화|불편 (매번 입력)|편함 (키 등록 후 자동)|
|브루트포스|취약|안전|
|설정 복잡도|간단|약간 복잡|

---

---

# ② OpenSSH 서버 설치 & 상태 확인

```bash
# 설치
sudo apt-get update
sudo apt-get install -y openssh-server

# 상태 확인
sudo systemctl status ssh
# active (running) ← 정상

# 시작 / 중지 / 재시작
sudo systemctl start ssh
sudo systemctl stop ssh
sudo systemctl restart ssh

# 부팅 시 자동 시작
sudo systemctl enable ssh
```

---

---

# ③ SSH 기본 접속 ⭐️

## 비밀번호 로그인

```bash
# 기본 접속
ssh 사용자@IP주소
ssh sshuser@192.168.1.10

# 포트 지정 (기본 22가 아닐 때)
ssh -p 2222 사용자@IP주소

# 처음 접속 시 호스트 키 확인 질문
# → yes 입력
# Are you sure you want to continue connecting? yes
```

## 원격 단일 명령 실행 ⭐️

```bash
# 접속하지 않고 명령어 하나만 실행
ssh 사용자@IP주소 "명령어"

ssh sshuser@127.0.0.1 "hostname"
ssh sshuser@127.0.0.1 "df -h"
ssh sshuser@127.0.0.1 "ls -l /"
# → 명령 실행 후 SSH 자동 종료 (exit 불필요)
```

```
자동화 활용:
  여러 서버 상태 동시 점검
  for server in 서버1 서버2 서버3; do
      ssh user@$server "free -h"
  done
```

## IP 주소 확인

```bash
ip addr              # 전체 네트워크 인터페이스 정보
hostname -I          # IP 주소만 깔끔하게 출력
# 172.16.50.3 172.17.0.1

# SSH 연결 정보 확인 (접속 후)
echo $SSH_CONNECTION
# 클라이언트IP 포트 서버IP 포트
```

## SSH 세션 종료

```bash
exit     # 세션 종료 후 원래 터미널로 복귀
logout   # 동일
Ctrl + D # 동일
```

---

---

# ④ 공개키 인증 개념 ⭐️

```
비대칭 암호화 방식:
  개인키 (id_rsa)     → 내 컴퓨터에만 보관 (절대 유출 금지)
  공개키 (id_rsa.pub) → 서버에 등록 (유출돼도 무관)

인증 과정:
  1. 서버: authorized_keys 에 공개키 등록됨
  2. 클라이언트: 개인키로 서명
  3. 서버: 공개키로 서명 검증 → 일치하면 인증 성공
  4. 비밀번호 입력 없이 접속
```

---

---

# ⑤ ssh-keygen — 키 쌍 생성 ⭐️

```bash
# RSA 키 쌍 생성
ssh-keygen -t rsa

# 상세 옵션
ssh-keygen -t rsa -b 4096           # 4096비트 (더 강력)
ssh-keygen -t ed25519               # Ed25519 (최신 권장)
ssh-keygen -t rsa -C "내 이메일"    # 주석 추가

# 실행 시 질문:
# 저장 경로: ~/.ssh/id_rsa (Enter 로 기본값)
# passphrase: 개인키 보호 암호 (빈칸도 가능, 보안상 설정 권장)
```

```bash
# 생성된 파일 확인
ls -l ~/.ssh/

# -rw------- 1 user user 2655 ... id_rsa      ← 개인키 (600 권한 필수)
# -rw-r--r-- 1 user user  574 ... id_rsa.pub  ← 공개키 (배포 가능)
```

```
파일 구분:
  id_rsa     = 개인키 (Private Key)
               → 소유자만 읽기 가능 (권한 600)
               → 절대 공유 금지
  id_rsa.pub = 공개키 (Public Key)
               → 서버에 등록하는 파일
               → 유출돼도 무관
```

---

---

# ⑥ ssh-copy-id — 공개키 서버에 등록 ⭐️

```bash
# 공개키를 서버의 authorized_keys 에 자동 추가
ssh-copy-id 사용자@서버IP

ssh-copy-id sshuser@localhost
ssh-copy-id sshuser@192.168.1.10

# 포트 지정
ssh-copy-id -p 2222 사용자@서버IP
```

```
ssh-copy-id 가 하는 일:
  로컬의 ~/.ssh/id_rsa.pub 를 읽어
  서버의 ~/.ssh/authorized_keys 에 추가
  ~/.ssh 디렉토리 없으면 자동 생성 + 권한 설정

수동으로 하려면:
  cat ~/.ssh/id_rsa.pub | ssh 사용자@서버 "cat >> ~/.ssh/authorized_keys"
```

## authorized_keys 확인

```bash
cat ~/.ssh/authorized_keys
# ssh-rsa AAAA... user@hostname
# ↑ 공개키 한 줄이 등록됨
# 여러 키 등록 가능 (한 줄에 하나씩)
```

---

---

# ⑦ 공개키 인증으로 접속

```bash
# 공개키 등록 후 접속
ssh sshuser@localhost
# passphrase 입력 (계정 비밀번호 아님 — 개인키 보호 암호)

# 접속 확인
whoami          # sshuser
echo $SSH_CONNECTION   # 연결 정보

exit
```

```
passphrase vs 비밀번호:
  비밀번호  = 계정 로그인 암호
  passphrase = 개인키 파일 자체를 보호하는 암호
               → 개인키가 도난당해도 passphrase 없으면 사용 불가
```

---

---

# ⑧ SSH 파일 전송 — scp

```bash
# 로컬 → 원격 복사
scp 파일.txt 사용자@서버:/경로/
scp data.csv sshuser@192.168.1.10:/home/sshuser/

# 원격 → 로컬 복사
scp 사용자@서버:/경로/파일.txt .

# 디렉토리 전체 복사
scp -r 디렉토리/ 사용자@서버:/경로/

# 포트 지정
scp -P 2222 파일.txt 사용자@서버:/경로/
```

---

---

# ⑨ SSH 설정 파일 — ~/.ssh/config

```bash
# ~/.ssh/config 작성 → 접속 단축키 설정
nano ~/.ssh/config
```

```
Host myserver                    # 별칭
    HostName 192.168.1.10        # 서버 IP
    User sshuser                 # 사용자명
    Port 22                      # 포트
    IdentityFile ~/.ssh/id_rsa   # 개인키 경로

Host prod
    HostName 10.0.0.100
    User ubuntu
    IdentityFile ~/.ssh/prod_key
```

```bash
# config 설정 후 단축 접속
ssh myserver          # ssh sshuser@192.168.1.10 과 동일
ssh prod
scp file.txt myserver:/home/
```

---

---

# ⑩ 보안 강화 — /etc/ssh/sshd_config

```bash
sudo nano /etc/ssh/sshd_config

# 주요 보안 설정:
PermitRootLogin no              # root 직접 접속 차단 (권장)
PasswordAuthentication no       # 비밀번호 로그인 차단 (공개키만)
PubkeyAuthentication yes        # 공개키 인증 허용
Port 2222                       # 기본 포트 변경 (선택)
AllowUsers sshuser ubuntu       # 특정 사용자만 허용

# 설정 변경 후 재시작
sudo systemctl restart ssh
```

```
실무 보안 원칙:
  root SSH 접속 차단 (PermitRootLogin no)
  비밀번호 로그인 차단 (PasswordAuthentication no)
  공개키만 허용
  → 브루트포스 공격 차단
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|Permission denied|권한 문제|`chmod 700 ~/.ssh` / `chmod 600 ~/.ssh/id_rsa`|
|공개키 인증 안 됨|authorized_keys 없음|`ssh-copy-id` 실행|
|passphrase 잊음|개인키 보호 암호 분실|새 키 쌍 생성 후 재등록|
|접속할 때마다 passphrase 입력|ssh-agent 없음|`ssh-add ~/.ssh/id_rsa`|
|포트 22 막힘|방화벽|`sudo ufw allow 22`|

## SSH 권한 체크리스트

```bash
chmod 700 ~/.ssh                   # .ssh 디렉토리
chmod 600 ~/.ssh/id_rsa            # 개인키
chmod 644 ~/.ssh/id_rsa.pub        # 공개키
chmod 600 ~/.ssh/authorized_keys   # authorized_keys
chmod 600 ~/.ssh/config            # config 파일
```