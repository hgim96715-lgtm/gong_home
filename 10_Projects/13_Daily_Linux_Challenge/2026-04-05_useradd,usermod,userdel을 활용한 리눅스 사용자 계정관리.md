---
tags:
  - Linux_Test
related:
  - "[[00_Linux_Challenge_DashBoard]]"
  - "[[Linux_Filesystem]]"
  - "[[Linux_User_Group]]"
source: Linux Journey
difficulty:
  - CompTIA Linux+ 자격증 취득 실습 랩
---
## 📌 랩 목표

- Linux 환경에서 사용자 계정 생성, 수정, 삭제의 전 과정을 익힌다.
- 계정 보안을 위한 비밀번호 정책(Lock, Expiration) 설정 방법을 습득한다.
- `su`와 `su -`의 차이점을 이해하여 환경 변수 관리 능력을 배양한다.
- 그룹 멤버십 관리를 통해 시스템 권한 제어 방식을 학습한다.

---

## 📖 학습 내용

---

### 🔷 섹션 1 —  `useradd` 와 `passwd` 를 이용한 사용자 생성 및 보안 설정

**개념 요약**

- **홈 디렉토리 생성 (`-m`):** `useradd` 실행 시 `-m` 옵션은 필수입니다. 이를 통해 사용자의 전용 공간인 `/home/사용자명`이 생성됩니다. 이 옵션이 없으면 계정은 존재하지만 작업 공간이 없는 상태가 됩니다.
- `/etc/passwd`: 계정의 기본 정보(UID, GID, 홈 디렉토리 경로, 쉘 등)가 기록됩니다.
- `/etc/shadow`: 암호화된 비밀번호와 만료 정책이 담긴 보안 파일입니다. (일반 사용자 접근 불가)
- `/etc/group`: 사용자가 속한 그룹 정보를 관리합니다.

**명령어 / 문법**

```bash
sudo useradd -m student1
sudo grep ^student1 /etc/passwd /etc/shadow /etc/group
# /etc/passwd:student1:x:5001:5001::/home/student1:/bin/sh /etc/shadow:student1:!:20265:0:99999:7::: /etc/group:student1:x:5001:

sudo passwd student1
sudo grep ^student1 /etc/shadow # student1:$y$j9T$lUM1RtLPQdrCOHmaFf1po/$xqNw.5dz54yR9whxsID9teI28/BOyvKocK5dA9X7GoD:20265:0:99999:7:::
```

**실습 메모**
- 반드시 관리자 권한(`sudo`)이 필요함.
---

### 🔷 섹션 2 —  `su` 와 `su -`를 통한 사용자 전환 및 환경 차이 이해

**개념 요약**

- **`su` (Substitute User):** 현재 사용자의 환경 변수(PATH, Shell 설정 등)를 유지한 채 계정만 바꿉니다. 작업 디렉토리도 변하지 않습니다.
- **`su -` (Login Shell):** 대상 사용자로 **로그인**하는 것과 동일한 효과를 냅니다. 해당 사용자의 프로필(`~/.bashrc` 등)을 새로 읽어오며, 홈 디렉토리로 자동 이동합니다.

**명령어 / 문법**

```bash
whoami echo $HOME pwd # labex /home/labex /home/labex/project

su student1
whoami echo $HOME pwd # student1 /home/student1 /home/labex/project
exit

su - student1
whoami echo $HOME pwd # student1 /home/student1 /home/student1
exit #logout
```

**실습 메모**

- `su`와 `su -` 실행 후 `pwd`와 `echo $HOME` 결과가 어떻게 다른지 비교하는 것이 핵심.

---

### 🔷 섹션 3 —   `passwd -l`과 `passwd -u`를 이용한 사용자 계정 잠금 및 해제

**개념 요약**

- **계정 잠금 (`-l`):** 비밀번호 해시 앞에 `!`를 추가하여 어떤 비밀번호를 입력해도 인증이 실패하게 만듭니다. (임시 정지 상태)
- **계정 해제 (`-u`):** 추가된 `!`를 제거하여 다시 정상적인 로그인이 가능하도록 복구합니다.

**명령어 / 문법**

```bash
sudo passwd -l student1 # passwd: password for user student1 changed.
sudo grep ^student1 /etc/shadow # student1:!$y$j9T$...:20265:0:99999:7:::

su - student1 # Password: su: Authentication failure

sudo passwd -u student1 # passwd: password for user student1 changed.
sudo grep ^student1 /etc/shadow # student1:$y$j9T$...:20265:0:99999:7:::
```

**실습 메모**

- 잠금 상태에서는 `su -` 시 정확한 비밀번호를 입력해도 `Authentication failure`가 뜨는 것을 확인함.

---
### 🔷 섹션 4 —    `chage` 와 `usermod` 를 이용한 비밀번호 만료 및 그룹 멤버십 수정

**개념 요약**

- **비밀번호 정책 (`chage`):** 보안 강화를 위해 비밀번호의 유효 기간, 최소 변경 주기, 만료 전 경고 일수를 설정합니다.
- **그룹 수정 (`usermod`):** * `-G`: 보조 그룹을 설정합니다. (기존 보조 그룹 삭제 주의)
- `-aG`: 기존 그룹을 유지하면서 새로운 그룹을 **추가(append)** 합니다.

**명령어 / 문법**

```bash
sudo chage -l student1 # Last password change : Dec 08, 2024 Password expires : never Password inactive : never Account expires : never Minimum number of days between password change : 0 Maximum number of days between password change : 99999 Number of days of warning before password expires : 7
sudo chage -M 90 -m 7 -W 14 student1 
# Last password change : Dec 08, 2024 Password expires : Mar 08, 2025 Password inactive : never Account expires : never Minimum number of days between password change : 7 Maximum number of days between password change : 90 Number of days of warning before password expires : 14


id student1 # uid=5001(student1) gid=5001(student1) groups=5001(student1)
sudo groupadd developers
sudo usermod -aG developers student1
id student1 # uid=5001(student1) gid=5001(student1) groups=5001(student1),1002(developers)

sudo groupadd testers sudo usermod -G testers student1
id student1 # uid=5001(student1) gid=5001(student1) groups=5001(student1),1003(testers)

sudo usermod -aG developers student1
id student1
uid=5001(student1) gid=5001(student1) groups=5001(student1),1003(testers),1002(developers)
```

**실습 메모**
- `usermod -G`만 사용했을 때 기존 그룹에서 튕겨 나가는 현상 주의. 실무에서는 무조건 `-aG` 사용 습관 들이기.

---
### 🔷 섹션 5 —     `userdel` 과 `userdel -r`을 이용한 사용자 및 데이터 삭제

**개념 요약**

- **단순 삭제 (`userdel`):** 시스템 계정 정보만 삭제하고 사용자가 쓰던 파일(홈 디렉토리)은 그대로 남겨둡니다.
- **완전 삭제 (`-r`):** 계정 정보와 함께 홈 디렉토리, 메일함까지 시스템에서 완전히 제거합니다.

**명령어 / 문법**

```bash
sudo useradd -m student2
sudo userdel student1
grep ^student1 /etc/passwd #  아무런 출력도 없음
ls -ld /home/student1 # drwxr-x--- 2 5001 5001 78 Jun 26 08:18 /home/student1

sudo userdel -r student2 # userdel: student2 mail spool (/var/mail/student2) not found
grep ^student2 /etc/passwd
ls -ld /home/student2 # ls: cannot access '/home/student2': No such file or directory
```

**실습 메모**
- `-r` 옵션 없이 삭제한 경우, `/home` 디렉토리에 남은 잔재를 수동으로 지워야 하는 번거로움이 있음.
---

## 🔴 헷갈렸던 것 / 새로 알게 된 것