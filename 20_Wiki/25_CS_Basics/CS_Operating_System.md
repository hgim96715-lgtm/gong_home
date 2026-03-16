---
aliases:
  - 운영체제
  - OS
  - Operating System
  - 프로세스
  - 메모리 관리
  - 스케줄링
tags:
  - CS
related:
  - "[[00_CS_HomePage]]"
  - "[[Linux_Process]]"
  - "[[Linux_Memory]]"
  - "[[Linux_Filesystem]]"
  - "[[Linux_Background_Jobs]]"
---

# CS_Operating_System — 운영체제 개념

## 한 줄 요약

```
운영체제 = 하드웨어와 소프트웨어 사이의 중간 관리자
하드웨어 자원(CPU / 메모리 / 디스크 / 네트워크)을
여러 프로그램이 안전하게 공유할 수 있도록 관리
```

---

---

# ① OS 의 역할

```
1. 자원 관리
   CPU / 메모리 / 디스크 / 네트워크를 프로세스에 할당

2. 하드웨어 추상화
   개발자가 하드웨어 세부사항 몰라도 되게 API 제공
   → 파일 읽기: read() 시스템 콜 하나로 끝

3. 보안 / 보호
   프로세스끼리 메모리 침범 못 하게 격리

4. 사용자 인터페이스
   CLI (bash) / GUI (GNOME) 제공
```

---

---

# ② 프로세스 vs 스레드

```
프로세스 (Process)
  실행 중인 프로그램의 독립된 인스턴스
  자체 메모리 공간 (코드 / 힙 / 스택)
  프로세스끼리 메모리 공유 안 함 → 격리
  생성 비용 높음 (fork)

스레드 (Thread)
  프로세스 안에서 실행되는 더 작은 실행 단위
  같은 프로세스 안의 스레드는 메모리 공유
  생성 비용 낮음
  → 공유 때문에 동기화 문제 발생 가능 (Race Condition)
```

```python
# Python 에서 프로세스 vs 스레드
import multiprocessing   # 프로세스 기반 (독립 메모리)
import threading         # 스레드 기반 (메모리 공유)

# Kafka Producer / Spark Worker = 별도 프로세스
# GIL (Python) → CPU 작업은 멀티프로세싱이 효율적
```

```
프로세스 상태 전이:
  생성(New) → 준비(Ready) → 실행(Running) → 대기(Waiting) → 종료(Terminated)

  Running → Waiting: I/O 요청 (디스크 읽기 등)
  Running → Ready:   타임슬라이스 만료 (선점 스케줄링)
```

---

---

# ③ CPU 스케줄링

```
여러 프로세스가 CPU 를 나눠 쓰는 방법
어떤 프로세스를 언제 실행할지 결정
```

|알고리즘|방식|특징|
|---|---|---|
|FCFS|먼저 온 순서|단순, Convoy Effect 발생|
|SJF|짧은 작업 먼저|평균 대기 최소, 기아(Starvation) 발생|
|Round Robin|시간 단위(quantum) 로 순환|공평, 시분할 OS 기본|
|Priority|우선순위 높은 순|기아 발생 → Aging 으로 해결|
|MLFQ|다단계 큐|현대 OS 실제 사용 방식|

```
리눅스 스케줄러:
  CFS (Completely Fair Scheduler)
  모든 프로세스에 공평한 CPU 시간 배분
  nice 값으로 우선순위 조정 (-20 ~ 19, 낮을수록 높은 우선순위)
```

---

---

# ④ 메모리 관리

## 가상 메모리 (Virtual Memory)

```
각 프로세스가 자신만의 연속된 메모리 공간을 가진 것처럼 착각하게 만듦
실제 물리 메모리(RAM) 보다 큰 공간을 사용 가능

가상 주소 → 페이지 테이블 → 물리 주소 변환
```

## 페이징 (Paging)

```
메모리를 고정 크기(페이지, 보통 4KB) 로 나눠서 관리
외부 단편화 제거
내부 단편화 소량 발생 가능

페이지 교체 알고리즘:
  FIFO  가장 오래된 페이지 교체
  LRU   가장 오래 안 쓴 페이지 교체 (현대 OS 기본)
  OPT   미래 참조 기반 (이론상 최적)
```

## 스와핑 (Swapping)

```
RAM 이 부족하면 디스크의 swap 영역으로 페이지 이동
→ 성능 급락 (디스크 I/O)
→ free -h 에서 swap 사용률 높으면 RAM 부족 신호

Linux:
  /proc/sys/vm/swappiness  스왑 사용 적극성 (0~100, 기본 60)
```

---

---

# ⑤ 파일시스템

## inode

```
리눅스 파일시스템의 핵심 자료구조
파일의 메타데이터 저장 (크기, 권한, 소유자, 타임스탬프, 데이터 블록 위치)
파일 이름은 inode 에 없음 → 디렉토리가 "이름 → inode 번호" 매핑

ls -i 로 inode 번호 확인 가능
```

```bash
ls -i file.txt        # inode 번호 출력
stat file.txt         # inode 상세 정보
df -i                 # inode 사용률 확인
```

## 파일시스템 종류

```
ext4    리눅스 기본 (저널링 지원)
XFS     대용량 파일 / RHEL 기본
NTFS    Windows
FAT32   USB / 임베디드
tmpfs   RAM 기반 임시 파일시스템 (/tmp)
```

---

---

# ⑥ 시스템 콜 (System Call)

```
사용자 프로그램이 OS 커널 기능을 요청하는 인터페이스
사용자 모드 → 커널 모드 전환 발생

자주 쓰이는 시스템 콜:
  파일: open / read / write / close
  프로세스: fork / exec / wait / exit
  메모리: mmap / brk
  네트워크: socket / connect / send / recv
```

```
인터럽트 (Interrupt):
  하드웨어가 CPU 에 "나 처리해줘" 신호
  키보드 입력 / 타이머 / 디스크 I/O 완료

  소프트웨어 인터럽트 = 시스템 콜
  하드웨어 인터럽트 = 외부 장치 신호
```

---

---

# ⑦ 데드락 (Deadlock)

```
두 개 이상의 프로세스가 서로 상대방이 가진 자원을 기다리며 영원히 멈춘 상태

발생 조건 (4가지 모두 만족해야 발생):
  1. 상호 배제 (Mutual Exclusion)   자원을 한 번에 하나만 사용
  2. 점유 대기 (Hold and Wait)      자원 가진 채로 다른 자원 대기
  3. 비선점 (No Preemption)         강제로 빼앗을 수 없음
  4. 환형 대기 (Circular Wait)      대기가 순환 구조

해결 방법:
  예방 (Prevention)    4가지 조건 중 하나 제거
  회피 (Avoidance)     은행원 알고리즘
  탐지 후 회복         교착 감지 → 프로세스 강제 종료
```

---

---

# ⑧ 리눅스에서 이론이 실제로 보이는 곳

| OS 개념     | 리눅스 명령어 / 파일                            |
| --------- | --------------------------------------- |
| 프로세스 상태   | `ps aux` → STAT 컬럼 (R/S/D/Z)            |
| 스케줄링 우선순위 | `nice` / `renice` / `top` → NI 컬럼       |
| 가상 메모리    | `cat /proc/meminfo` / `vmstat`          |
| 스와핑       | `free -h` → Swap 줄                      |
| inode     | `ls -i` / `df -i` / `stat`              |
| 시스템 콜 추적  | `strace` 명령어                            |
| 데드락 탐지    | `deadlock detector` / `lsof` 로 파일 잠금 확인 |
| 인터럽트      | `cat /proc/interrupts`                  |