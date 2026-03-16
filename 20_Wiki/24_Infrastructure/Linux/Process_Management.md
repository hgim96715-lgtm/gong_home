---
aliases:
  - ps
  - top
  - htop
  - kill
  - PID
  - 프로세스 관리
tags:
  - Linux
related:
  - "[[Linux_Signals]]"
  - "[[Background_Jobs]]"
  - "[[00_Linux_HomePage(기존)]]"
  - "[[Time_Synchronization]]"
---
##  개념 한 줄 요약

**"작업 관리자(Task Manager)를 터미널에서 켜보자."**

* **`ps` (Snapshot):** 현재 시점의 프로세스 목록을 **사진 찍듯이** 찰칵 보여준다. (정적)
* **`top` (Real-time):** 작업 관리자처럼 실시간으로 **움직이는 현황**을 보여준다. (동적)
* **`kill` (Action):** 문제가 된 프로세스를 **종료**시킨다.

---
## 왜 필요한가? (Why)

**문제점:**
- "서버가 갑자기 엄청 느려졌어요!"
- "파이썬 코드를 돌려놨는데, 이게 죽었는지 살았는지 모르겠어요."

**해결책:**
- **`top`** 을 켜서 누가 CPU를 100% 쓰고 있는지 범인을 색출한다.
- **`ps`** 로 그 범인의 **PID(주민번호)** 를 알아낸다.
- **`kill`** 로 PID를 저격해서 종료시킨다.

---
## 3. 핵심 개념: PID (Process ID) 

리눅스에서 실행 중인 모든 프로그램은 **고유한 번호(PID)** 를 가진다.
우리가 프로그램을 종료하거나 관리할 때는 이름이 아니라 **무조건 이 번호(PID)** 를 사용한다.

---
## Code Core Points: ① 감시하기 (`ps`, `top`)

### A. `ps`: 찰칵! 순간 포착 

가장 많이 쓰는 옵션은 **`aux`** 또는 **`ef`**다

```bash
# 1. 모든 프로세스 자세히 보기 (국룰 명령어)
# a(All users), u(User detail), x(No terminal)
ps aux

# 2. 특정 프로세스만 콕 집어서 찾기 (grep 조합) ⭐️
# "python 들어간 거 다 찾아봐"
ps aux | grep python
```

**💡 `ps aux` 결과 항목 보는 법**

- **USER:** 누가 실행했나?
- **PID:** 프로세스 번호 (이걸 알아야 `kill`을 함)
- **%CPU, %MEM:** 자원 사용량
- **COMMAND:** 실행된 명령어 (무슨 프로그램인지 확인)

```text
ubuntu@d16b67ad72a3:~$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
ubuntu       1  0.0  0.0   2088   888 ?        Ss   03:47   0:00 /usr/bin/dumb-init -- sudo /root/bootstrap.sh
root         7  0.0  0.0   9456  4048 pts/0    Ss+  03:47   0:00 sudo /root/bootstrap.sh
ubuntu     141  0.0  0.0   4144  3352 pts/1    Ss   08:49   0:00 bash
ubuntu     232  0.0  0.0   6444  2440 pts/1    R+   13:24   0:00 ps aux
```

**결과 표**

|컬럼|의미|
|---|---|
|**USER**|프로세스를 실행한 사용자|
|**PID**|프로세스 ID (kill 대상)|
|**%CPU**|CPU 사용률|
|**%MEM**|메모리 사용률|
|**VSZ**|가상 메모리 크기 (KB)|
|**RSS**|실제 사용 중인 물리 메모리 (KB) ⭐|
|**TTY**|연결된 터미널 (`?` = 터미널 없음, 데몬/백그라운드)|
|**STAT**|프로세스 상태|
|**START**|프로세스 시작 시간|
|**TIME**|누적 CPU 사용 시간|
|**COMMAND**|실행된 명령어|
**STAT 자주 보는 값**

|값|의미|
|---|---|
|**R**|실행 중 (Running)|
|**S**|대기 중 (Sleeping)|
|**D**|I/O 대기 (디스크 문제 가능 ⚠️)|
|**Z**|좀비 프로세스 🚨|
|**+**|포그라운드 프로세스|
|**s**|세션 리더|
`Ss` → 대기 중 + 세션 리더  / `R+` → 실행 중 + 포그라운드



### B. `top`: 실시간 생중계 

윈도우의 '작업 관리자' 화면과 똑같다. 3초마다 갱신된다.

```bash
# 실행
top

# [top 실행 중 단축키]
# q : 나가기 (Quit)
# 1 : CPU 코어별로 쪼개서 보기
# P : CPU 많이 쓰는 순서로 정렬 (기본값)
# M : 메모리 많이 쓰는 순서로 정렬
# k : 프로세스 죽이기 (PID 입력하라고 뜸)
```

**결과**

```text
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
1   ubuntu    20   0    2088     888    804 S   0.0   0.0   0:00.00 dumb-init
# ↑ PID 1 : 컨테이너/시스템의 시작 프로세스 (모든 프로세스의 조상)

7   root      20   0    9456    4048   3468 S   0.0   0.0   0:00.00 sudo
# ↑ sudo로 실행된 부모 작업

141 ubuntu    20   0    4144    3352   2796 S   0.0   0.0   0:00.40 bash
# ↑ 내가 쓰고 있는 쉘 (터미널)

235 ubuntu    20   0    6744    2776   2292 R   0.0   0.0   0:00.00 top
# ↑ 현재 실행 중인 top (R = Running)
```

**컬럼 요약**

|컬럼|의미|
|---|---|
|**PID**|프로세스 번호 (kill 대상)|
|**USER**|프로세스를 실행한 사용자|
|**PR / NI**|우선순위 (일반적으로 신경 안 써도 됨)|
|**VIRT**|프로세스가 사용하는 가상 메모리 전체|
|**RES**|실제 RAM에서 사용 중인 메모리 ⭐|
|**SHR**|다른 프로세스와 공유 중인 메모리|
|**S**|프로세스 상태 (R=실행, S=대기)|
|**%CPU**|CPU 점유율|
|**%MEM**|메모리 점유율|
|**TIME+**|누적 CPU 사용 시간|
|**COMMAND**|실행 중인 명령|


**꿀팁:** 요즘엔 `top`보다 예쁘고 마우스도 되는 **`htop`**을 더 많이 쓴다. (`apt install htop` 필요)

---
## Code Core Points: ② 처단하기 (`kill`)

범인의 PID를 알아냈으면 종료시킨다. _(자세한 종료 방식(시그널)은 [[Linux_Signals]] 문서를 참고할 것)_

```bash
# 기본 문법: kill [PID]

# 1. 정중하게 종료 요청 (SIGTERM - 15번)
# 대부분 이걸로 해결됨.
kill 1234

# 2. 강제 종료 (SIGKILL - 9번) 
# 말이 안 통하는 녀석(멈춤)을 강제로 끌어내림.
kill -9 1234
```

---
## 실무 적용 시나리오 (Workflow)

**상황:** 서버가 느리다. 범인을 찾아서 죽여라.

**Step 1. 범인 색출 (`top`)**

```bash
top
# 화면을 보니 'python3 mining_script.py'가 CPU 99%를 먹고 있다.
# PID가 '5678'이라는 것을 확인했다.
# 'q'를 눌러서 나온다.
```

Step 2. 확인 사살 (`ps`)

```bash
# 혹시 모르니 다시 한번 확인한다.
ps aux | grep 5678
```

Step 3. 처형 (`kill`)

```bash
# 일단 점잖게 종료 요청
kill 5678

# (다시 ps 해봤는데 안 죽었으면?)
# 강제 종료!
kill -9 5678
```

---
## 심화: 데몬 (Daemon) 이란? 

### ① 개념

* **정의:** 사용자가 직접 제어하지 않아도, 백그라운드에서 **24시간 조용히 돌면서 특정 요청을 기다리는 프로세스**입니다.
* **이름의 특징:** 보통 이름 끝에 **`d`** 가 붙습니다.
    * `sshd`: SSH 접속을 기다리는 데몬
    * `httpd`: 웹 페이지 요청을 기다리는 데몬
    * `chronyd`: 시간 동기화를 담당하는 데몬
    * `dockerd`: 도커 컨테이너를 관리하는 데몬

### ② 관리 방법 (systemctl)

데몬을 켜고 끄는 건 `systemctl` 명령어로 합니다. (일반 프로세스처럼 `kill`로 잘 안 죽입니다.)

```bash
# 상태 확인 (살아있니?)
sudo systemctl status sshd

# 시작 / 중지 / 재시작
sudo systemctl start sshd
sudo systemctl stop sshd
sudo systemctl restart sshd

# 부팅 시 자동 실행 등록 (Enable) ⭐️
# "서버 껐다 켜도 자동으로 실행돼야 해!"
sudo systemctl enable sshd
```

> docker 을 쓸때 daemon 을 키는 방법은 조금 다름 [[Time_Synchronization#트러블슈팅 "506 Cannot talk to daemon"|daemon을 못찾을때]]


---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`kill` 명령어 쳤는데 안 죽어요!"

- **이유 1:** 권한이 부족해서. (`sudo kill ...` 해야 함)
- **이유 2:** 프로그램이 너무 바빠서 종료 신호를 못 들음. (이때 `kill -9`를 씀)


### ② "`top`에서 못 나오겠어요 ㅠㅠ"

- `Ctrl+C`를 눌러도 되지만, 정석은 **`q` (Quit)** 키를 누르는 것이다.

### ③ "PID가 계속 바뀌어요!"

- 프로그램이 죽었다가 다시 살아나고(재시작) 있을 수 있다.
- 이건 누군가(서비스 매니저, 도커 등)가 살려내고 있는 것이니, 근본적인 설정을 꺼야 한다.