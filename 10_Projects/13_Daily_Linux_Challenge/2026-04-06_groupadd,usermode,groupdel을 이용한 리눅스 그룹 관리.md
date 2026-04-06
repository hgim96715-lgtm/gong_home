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

리눅스 그룹 생성, 사용자 추가, 권한 확인 및 삭제 프로세스 습득

---

## 📖 학습 내용

---

### 🔷 섹션 1 — groupadd 로 새로운 리눅스 그룹 생성하기

**개념 요약**

`groupadd`: 새로운 그룹을 생성하는 명령어
`/etc/group`: 그룹 정보가 저장되는 파일 (`그룹명:비밀번호:GID:멤버` 형식)

**명령어 / 문법**

```bash
sudo groupadd research
grep research /etc/group # research:x:5003:
```

**실습 메모**

- `sudo` 권한 없이 실행하면 'permission denied' 발생. 반드시 관리자 권한 필요.

---

### 🔷 섹션 2 — usermod 로 보조 그룹에 사용자 추가하기

**개념 요약**

-  `-G`: 추가할 보조 그룹 지정
- `-a`: **Append(추가)** 의미. 기존 그룹 목록을 유지하며 새 그룹을 추가할 때 필수

**명령어 / 문법**

```bash
sudo usermod -aG research labex
grep research /etc/group # research:x:5003:labex
```

**실습 메모**

- **주의:** `-a` 옵션을 빼먹고 `sudo usermod -G research labex`라고 치면, `labex`가 기존에 속해있던 `sudo`, `docker` 등 다른 그룹에서 튕겨 나가게 됨. (매우 중요!)

----
### 🔷 섹션 3 — grep 과 groups 로 그룹 및 사용자 멤버십 확인하기

**개념 요약**

`groupdel`: 더 이상 필요 없는 그룹을 시스템에서 완전히 제거

**명령어 / 문법**

```bash
grep labex /etc/group
# sudo:x:27:labex ssl-cert:x:121:labex labex:x:5000: public:x:5002:labex research:x:5003:labex

groups labex # labex : labex sudo ssl-cert public research
```

**실습 메모**

- `groups` 명령어는 사용자의 현재 상태를 가장 빠르게 보여줌.

----
### 🔷 섹션 4 — groupdel 로 그룹 삭제 및 제거 확인하기

**개념 요약**

이는 팀이 해체되거나 프로젝트가 완료되어 관련 그룹이 더 이상 필요하지 않을 때 수행하는 일반적인 관리 작업입니다. 그룹을 삭제할 때는 `groupdel` 명령어를 사용합니다.

**명령어 / 문법**

```bash
sudo groupdel research
grep research /etc/group # 아무것도 나오지 않음 
```

**실습 메모**

- 삭제 후 `grep`으로 확인했을 때 아무 결과도 안 나오는 것이 '성공'의 증거.

---
---

## 🔴 헷갈렸던 것 / 새로 알게 된 것