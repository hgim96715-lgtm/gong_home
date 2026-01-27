---
aliases:
  - apt
  - 패키지관리
  - 프로그램설치
  - yum
  - dnf
  - update
  - upgreade
tags:
  - Linux
related:
  - "[[Disk_Management]]"
  - "[[Process_Management]]"
  - "[[00_Linux_HomePage]]"
---
##  개념 한 줄 요약

**"리눅스 세상의 '앱 스토어'이자 '설치 마법사'."**

* APT (Advanced Package Tool):** 우분투(Ubuntu), 데비안(Debian) 계열 리눅스에서 사용하는 패키지 관리 도구입니다
* 프로그램의 설치, 업데이트, 제거, 그리고 가장 중요한 **의존성(Dependency)** 관리까지 알아서 다 해줍니다. 

---

## 왜 필요한가? (Why)

**문제점:**
- "웹 서버(Nginx)를 깔고 싶은데, 관련된 라이브러리 파일 10개를 일일이 찾아서 먼저 깔아야 한대요." (의존성 지옥)
- "내가 깐 프로그램들의 버전을 최신으로 유지하고 싶은데, 하나씩 사이트 들어가서 다운받아야 하나요?"

**해결책:**
- **`apt install nginx`** 한 줄이면 필요한 라이브러리까지 **자동으로** 다 찾아서 설치해 줍니다
- **`apt upgrade`** 명령어로 설치된 모든 프로그램을 한 방에 최신 버전으로 업데이트합니다.

---

##  Code Core Points: 필수 명령어 3단계

리눅스 서버를 처음 받으면 무조건 이 순서대로 실행하는 게 국룰입니다.
(모든 명령어 앞에는 관리자 권한인 `sudo`가 필요합니다.)

### ① 목록 갱신 & 업그레이드 (Update & Upgrade) 

가장 헷갈리는 부분입니다. **update**가 설치가 아닙니다!

```bash
# 1. 패키지 리스트 갱신 (실제 업데이트 X, 명단만 최신화 O)
# "앱 스토어에 무슨 앱이 새로 나왔는지 목록 좀 가져와."
sudo apt update  

# 2. 실제 업그레이드 실행 (설치 O)
# "내 컴퓨터에 깔린 앱들 중에 구버전 있으면 다 새걸로 바꿔줘."
sudo apt upgrade
```

### ② 설치 & 삭제 (Install & Remove)

```bash
# 1. 프로그램 설치 (예: vim 에디터 설치)
# 필요한 의존성 파일들도 알아서 같이 설치됨.
sudo apt install vim

# 2. 프로그램 삭제
# 프로그램만 지우고 설정 파일은 남김.
sudo apt remove vim
```

### ③ 검색 & 정보 확인 (Search & Show)

```bash
# 1. 패키지 검색
# "editor라는 단어가 들어간 프로그램 뭐 있어?"
apt search editor

# 2. 패키지 상세 정보 확인
# "vim 버전이 몇이고, 누가 만들었어?"
apt show vim
```

---
## 상세 분석: 저장소(Repository)와 OS별 차이

### A. 저장소 (Repository)

APT는 인터넷 어딘가에 있는 **'중앙 저장소'** 에서 파일을 받아옵니다.
- 이 주소들은 **`/etc/apt/sources.list`** 파일에 적혀 있습니다.
- 만약 `apt install`이 안 된다면 이 주소가 막혀있거나 꼬인 경우가 많습니다.

### B. OS별 패키지 매니저 차이 (족보)

리눅스 종류마다 명령어가 다릅니다. 회사 가서 당황하지 않으려면 알아둬야 합니다

|**OS 종류**|**패키지 매니저**|**비고**|
|---|---|---|
|**Ubuntu / Debian**|**`apt`**|가장 대중적. (우리가 배우는 것)|
|**RedHat / CentOS**|**`yum`** 또는 **`dnf`**|서버 시장에서 많이 씀.|
|**Arch Linux**|**`pacman`**|고급 사용자용.|
|**macOS**|**`brew`** (Homebrew)|(참고) 맥은 리눅스가 아니라서 `brew`를 씀.|

---
## 보자가 자주 하는 실수 (Misconceptions)

### ① "`apt update` 했는데 왜 프로그램 버전이 그대로죠?"

- **Update**는 **"새로 나온 전단지(목록)"**만 받아오는 겁니다.
- 실제로 물건을 바꾸려면 **`apt upgrade`** 를 해야 합니다. 
- 그래서 보통 `sudo apt update && sudo apt upgrade` 이렇게 이어서 많이 씁니다.

### ② "`apt remove` 했는데 설정이 남아있어요!"

- `remove`는 실행 파일만 지웁니다. 
- 내가 열심히 세팅해둔 설정 파일(`config`)까지 싹 밀어버리고 싶다면 **`purge`** 를 써야 합니다.
    - `sudo apt purge vim`


### ③ "제 맥북 터미널에서 `apt`가 안 돼요."

- `apt`는 리눅스(우분투)용입니다. 
- 맥북에서는 **Homebrew (`brew`)** 를 설치해서 써야 합니다. (역할은 똑같습니다!)