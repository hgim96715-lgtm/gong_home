---
aliases:
  - 프로세스
  - ps
  - top
  - kill
  - signal
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Background_Jobs]]"
  - "[[Linux_Memory]]"
  - "[[CS_Operating_System]]"
---

# Linux_Process — 프로세스 관리

## 한 줄 요약

```
프로세스 = 실행 중인 프로그램
ps  → 특정 순간의 스냅샷
top → 실시간 모니터링
kill → 프로세스에 신호 보내기
```

---

---

# ① 프로세스 개념

```
프로세스 = 실행 중인 프로그램의 인스턴스
  프로그램: 디스크에 저장된 코드 (정적)
  프로세스: 메모리에 올라가 실행 중인 상태 (동적)

PID (Process ID):
  각 프로세스에 부여되는 고유 번호
  부팅 시 1번 (systemd/init) 부터 시작
  부모 프로세스가 자식 프로세스를 생성

PPID (Parent PID):
  부모 프로세스의 ID
  ps -ef 에서 확인 가능
```

## 프로세스 상태

```
R  Running    → 실행 중 / CPU 사용 중
S  Sleeping   → 대기 중 (이벤트 기다림)
D  Disk sleep → 디스크 I/O 대기 (인터럽트 불가)
Z  Zombie     → 종료됐지만 부모가 아직 회수 안 함
T  Stopped    → 일시 중지 (Ctrl-Z)
```

---

---

# ② ps — 프로세스 스냅샷 ⭐️

```
ps = Process Status
특정 순간의 프로세스 목록을 스냅샷으로 보여줌
실시간 아님
```

## 기본 사용

```bash
ps              # 현재 터미널의 프로세스만
ps aux          # 시스템 전체 + CPU/MEM 정보
ps -ef          # 시스템 전체 + 부모-자식 관계

# 특정 프로세스 찾기
ps aux | grep sleep
ps aux | grep python
ps -ef | grep airflow
```

## ps aux vs ps -ef 비교

```
ps aux:
  a  = 모든 사용자의 프로세스
  u  = 사용자 중심 형식 (CPU%, MEM% 포함)
  x  = 터미널 없는 프로세스 포함
  → 리소스(CPU, RAM) 확인할 때 유리

ps -ef:
  -e = 시스템 모든 프로세스
  -f = full format (PPID 포함)
  → 부모-자식 관계 파악할 때 유리
```

## ps aux 출력 해석

```bash
ps aux
# USER    PID   %CPU  %MEM  VSZ    RSS   TTY  STAT  START  TIME  COMMAND
# root      1    0.0   0.1  168M   13M   ?    Ss    Apr09  0:01  /sbin/init
# labex   482    0.0   0.0    0     0    pts/0 S     10:30  0:00  sleep 300

# %CPU   = CPU 사용률
# %MEM   = 메모리 사용률
# VSZ    = 가상 메모리 크기
# RSS    = 실제 물리 메모리 사용
# STAT   = 프로세스 상태 (S=sleeping, R=running, Z=zombie)
```

## 특정 컬럼만 출력

```bash
# pid, 나이스값, 명령어만 출력
ps -o pid,ni,cmd -p 482

# CPU 많이 쓰는 프로세스 TOP 10
ps aux --sort=-%cpu | head -10

# 메모리 많이 쓰는 프로세스 TOP 10
ps aux --sort=-%mem | head -10
```

---

---

# ③ top — 실시간 모니터링 ⭐️

```
ps = 특정 순간의 스냅샷
top = 실시간 스트리밍 (3초마다 자동 갱신)

서버가 갑자기 느려졌을 때
→ top 켜고 P/M 눌러서 범인 프로세스 즉시 파악
```

## 실행 및 단축키

```bash
top          # 실행
htop         # 더 보기 좋은 버전 (설치 필요)
```

```
top 단축키 (대소문자 구분):
  P   → CPU 사용량 순 정렬 (기본값)
  M   → 메모리 사용량 순 정렬
  k   → 프로세스 종료 (PID 입력)
  r   → renice (우선순위 변경)
  q   → top 종료
  ↑↓  → 목록 스크롤
  1   → CPU 코어별 사용량
```

## top 출력 해석

```
상단 요약:
  top - 10:30:00 up 2 days  → 서버 가동 시간
  Tasks: 95 total, 1 running, 94 sleeping  → 프로세스 현황
  %Cpu: 5.0 us, 1.0 sy, 0.0 ni, 94.0 id  → CPU 사용
    us = user / sy = system / id = idle(유휴)
  MiB Mem: 3800 total, 800 free, 1200 used → 메모리

프로세스 목록:
  PID    USER    PR  NI   VIRT   RES   SHR  S  %CPU  %MEM  TIME+   COMMAND
  482    labex   20   0      0     0     0  S   0.0   0.0   0:00   sleep
```

## htop vs top

```
top:
  모든 서버에 기본 설치
  기능 단순

htop:
  색상 / 마우스 지원 / 트리 보기
  별도 설치 필요: sudo apt-get install htop
  직관적으로 CPU/메모리 막대 그래프 확인
```

---

---

# ④ renice — 프로세스 우선순위 조정 ⭐️

```
niceness 값 = 다른 프로세스에 CPU 를 얼마나 양보하는가
  -20  가장 높은 우선순위 (이기적 / CPU 독점)
    0  기본값
  +19  가장 낮은 우선순위 (친절 / 양보)

비유:
  -20 → "내가 먼저야!" (새치기)
    0 → "보통대로 할게"
  +19 → "다들 먼저 하세요" (양보)
```

## 권한 규칙

```
일반 유저:
  자신의 프로세스만 조정 가능
  낮추기 (양보, 숫자 높이기) 만 가능

root:
  모든 프로세스 조정 가능
  높이기 (독점, 숫자 낮추기) 가능
  → 우선순위 독점은 시스템 전체를 느리게 할 수 있으므로 root 만
```

## 명령어

```bash
# 현재 나이스 값 확인
ps -o pid,ni,cmd -p 482

# 나이스 값 변경
renice -n 10 -p 482     # PID 482 의 niceness 를 10 으로
# 482 (process ID) old priority 0, new priority 10

renice -n 19 -p 482     # 가장 낮은 우선순위로

# root 권한으로 우선순위 높이기
sudo renice -n -10 -p 482

# 프로세스 시작 시 우선순위 지정
nice -n 10 python script.py   # niceness 10 으로 시작
```

---

---

# ⑤ kill — 프로세스에 신호 보내기 ⭐️

```
kill = 죽이는 게 아니라 "신호(signal)를 보내는" 명령어
프로세스가 신호를 받고 어떻게 반응할지는 프로세스가 결정
```

## signal 종류

|번호|이름|의미|
|---|---|---|
|1|SIGHUP|재시작 요청 (설정 파일 리로드)|
|2|SIGINT|인터럽트 (Ctrl-C 와 동일)|
|9|SIGKILL|강제 종료 (프로세스가 무시 불가)|
|15|SIGTERM|정상 종료 요청 (기본값)|
|19|SIGSTOP|일시 중지 (무시 불가)|
|20|SIGTSTP|일시 중지 (Ctrl-Z 와 동일)|

## 명령어

```bash
# SIGTERM (기본값) — 정상 종료 요청
kill 482          # PID 로
kill %1           # Job ID 로

# SIGKILL — 강제 종료 (최후의 수단)
kill -9 482
kill -9 %1
kill -KILL 482    # 동일

# SIGHUP — 설정 리로드 (Nginx/Apache 에서 자주 사용)
kill -1 482
kill -HUP 482

# Ctrl-C = SIGINT (실행 중 프로세스 종료)
# Ctrl-Z = SIGTSTP (실행 중 프로세스 일시 중지)
```

## SIGTERM vs SIGKILL 차이 ⭐️

```
SIGTERM (15) — 정중한 요청:
  "종료해주세요"
  프로세스가 받고 → 정리 작업 → 종료
  DB 연결 끊기 / 파일 저장 / 로그 기록 후 종료
  프로세스가 무시할 수 있음 (처리 중이면 거부 가능)

SIGKILL (9) — 강제 종료:
  OS 가 직접 프로세스를 즉시 제거
  프로세스가 거부 불가
  정리 작업 없이 강제 종료 → 데이터 손상 위험

원칙:
  먼저 kill (SIGTERM) 시도
  응답 없으면 kill -9 (SIGKILL) 사용
```

## killall / pkill — 이름으로 종료

```bash
# 같은 이름의 모든 프로세스 종료
killall sleep
killall python3

# 패턴으로 종료
pkill -f "python script.py"   # 명령어에 포함된 문자열로 찾아서 종료
pkill sleep
```

---

---

# ⑥ 프로세스 트리 — pstree

```bash
pstree              # 전체 트리 구조
pstree -p           # PID 포함
pstree labex        # 특정 유저의 프로세스 트리
pstree -p | grep sleep
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`kill -9` 먼저 사용|습관|먼저 `kill` (SIGTERM) 시도|
|ps 결과에 자기 자신 포함|grep 자체가 프로세스|`grep -v grep` 추가|
|niceness 낮추기 실패|일반 유저 권한|`sudo renice` 사용|
|Zombie 프로세스 kill 안 됨|이미 종료된 상태|부모 프로세스 종료해야 해결|

```bash
# grep 자체 제외 패턴
ps aux | grep sleep | grep -v grep
# 또는
ps aux | grep [s]leep   # 정규식으로 grep 자체 제외
```