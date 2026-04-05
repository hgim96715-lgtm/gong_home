---
aliases:
  - 사용자 관리
  - 그룹 관리
  - useradd
  - usermod
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_File_Permissions]]"
  - "[[Linux_Superuser]]"
  - "[[Linux_SSH]]"
---

# Linux_User_Group — 사용자 & 그룹 관리

## 한 줄 요약

```
리눅스는 다중 사용자 시스템
사용자와 그룹으로 "누가 무엇에 접근할 수 있는지" 제어
```

---

---

# ① 사용자 관련 파일 ⭐️

## /etc/passwd — 사용자 정보

```bash
cat /etc/passwd
# username:x:UID:GID:comment:home_dir:shell
# gong:x:1001:1001::/home/gong:/bin/bash
```

```
필드 설명:
  username   사용자명
  x          패스워드 (실제 값은 /etc/shadow 에 저장)
  UID        사용자 ID (root=0 / 일반=1000~)
  GID        기본 그룹 ID
  comment    설명 (보통 이름)
  home_dir   홈 디렉토리
  shell      기본 셸 (/bin/bash / /sbin/nologin 등)
```

## /etc/shadow — 암호화된 비밀번호

```bash
sudo cat /etc/shadow
# username:암호화된비밀번호:마지막변경일:최소사용일:최대사용일:...
```

```
root 만 읽을 수 있음
비밀번호는 해시로 저장 (SHA-512 등)
! 또는 * 로 시작하면 로그인 비활성화
```

## /etc/group — 그룹 정보

```bash
cat /etc/group
# groupname:x:GID:member1,member2
# docker:x:999:gong,ubuntu
```

---

---

# ② 사용자 생성 / 삭제

## useradd — 사용자 생성

```bash
# 기본 생성
sudo useradd newuser

# 옵션과 함께 생성
sudo useradd -m -s /bin/bash -c "설명" newuser
```

|옵션|의미|
|---|---|
|`-m`|홈 디렉토리 생성|
|`-s /bin/bash`|기본 셸 지정|
|`-c "이름"`|설명 추가|
|`-g groupname`|기본 그룹 지정|
|`-G g1,g2`|추가 그룹 지정|
|`-u 1500`|UID 직접 지정|

```bash
# 실전 예시
sudo useradd -m -s /bin/bash -c "데이터 엔지니어" -G docker,sudo gong
```

## passwd — 비밀번호 설정

```bash
sudo passwd newuser     # 비밀번호 설정
passwd                  # 내 비밀번호 변경
```

## userdel — 사용자 삭제

```bash
sudo userdel newuser          # 사용자만 삭제 (홈 디렉토리 유지)
sudo userdel -r newuser       # 사용자 + 홈 디렉토리 함께 삭제
```

---

---

# ③ 사용자 수정

## usermod — 사용자 정보 변경

```bash
sudo usermod -s /bin/zsh gong           # 셸 변경
sudo usermod -d /new/home gong          # 홈 디렉토리 변경
sudo usermod -aG docker gong            # 그룹 추가 ← -a 필수!
sudo usermod -G docker,sudo gong        # 그룹 전체 교체
sudo usermod -l newname oldname         # 이름 변경
sudo usermod -L gong                    # 계정 잠금 (Lock)
sudo usermod -U gong                    # 계정 잠금 해제 (Unlock)
```

```
⚠️ -aG vs -G 차이:
  -aG docker  → 기존 그룹 유지 + docker 추가  ✅
  -G docker   → 기존 그룹 전부 제거 + docker 만 남김  ❌ 위험!

  그룹 추가할 땐 반드시 -aG (append + Group)
```

---

---

# ④ 그룹 생성 / 삭제 / 수정

```bash
sudo groupadd mygroup              # 그룹 생성
sudo groupadd -g 2000 mygroup      # GID 지정하며 생성
sudo groupdel mygroup              # 그룹 삭제
sudo groupmod -n newname oldname   # 그룹 이름 변경
```

---

---

# ⑤ 사용자 정보 확인

```bash
whoami                  # 현재 사용자명
id                      # UID / GID / 소속 그룹 전체
id gong                 # 특정 사용자 정보
groups                  # 현재 사용자 소속 그룹 목록
groups gong             # 특정 사용자 그룹 목록

who                     # 현재 로그인된 사용자
w                       # 로그인 사용자 + 작업 내용
last                    # 최근 로그인 이력
```

```bash
id gong
# uid=1001(gong) gid=1001(gong) groups=1001(gong),999(docker),27(sudo)
#  ↑UID          ↑기본GID        ↑소속된 모든 그룹
```

---

---

# ⑥ UID / GID 범위 규칙

```
0          root (최고 권한)
1 ~ 999    시스템 사용자 (데몬 / 서비스용)
1000 ~     일반 사용자 (사람이 쓰는 계정)

/sbin/nologin 셸:
  시스템 사용자는 로그인 불가
  mysql / nginx / kafka 등 서비스 계정에 사용
```

---

---

# ⑦ 실전 패턴

## Docker 그룹 추가 (sudo 없이 docker 쓰기)

```bash
sudo usermod -aG docker $USER
# 적용하려면 재로그인 또는
newgrp docker
```

## 서비스 전용 계정 생성

```bash
# 로그인 불가 / 홈 없는 서비스 계정
sudo useradd -r -s /sbin/nologin kafka
# -r  시스템 계정 (UID 1000 미만)
# -s /sbin/nologin  로그인 차단
```

## 사용자 생성 후 확인

```bash
sudo useradd -m -s /bin/bash gong
id gong
cat /etc/passwd | grep gong
ls /home/gong
```

---

---

# 한눈에 정리

|명령어|역할|
|---|---|
|`useradd -m -s /bin/bash 이름`|사용자 생성|
|`passwd 이름`|비밀번호 설정|
|`userdel -r 이름`|사용자 + 홈 삭제|
|`usermod -aG 그룹 이름`|그룹 추가|
|`usermod -L / -U 이름`|잠금 / 해제|
|`groupadd 그룹명`|그룹 생성|
|`groupdel 그룹명`|그룹 삭제|
|`id 이름`|사용자 UID/GID/그룹 확인|
|`groups 이름`|소속 그룹 확인|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`usermod -G docker` 후 기존 그룹 사라짐|`-a` 없이 `-G` 사용|`usermod -aG docker 이름`|
|`useradd` 후 홈 디렉토리 없음|`-m` 옵션 빠짐|`useradd -m 이름`|
|그룹 추가 후 적용 안 됨|재로그인 필요|`newgrp 그룹명` 또는 재로그인|
|사용자 삭제 후 파일 남음|`-r` 없이 삭제|`userdel -r 이름`|
|docker 그룹인데 sudo 필요|그룹 적용 안 됨|재로그인 또는 `newgrp docker`|