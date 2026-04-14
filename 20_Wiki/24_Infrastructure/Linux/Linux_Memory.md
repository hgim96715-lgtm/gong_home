---
aliases:
  - 리눅스 메모리
  - swap
  - free
  - vmstat
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Disk]]"
  - "[[Linux_Process]]"
---

# Linux_Memory — 메모리 & 스왑 관리

## 한 줄 요약

```
free     → 메모리 / 스왑 사용량 확인
swap     → RAM 부족 시 사용하는 가상 메모리 공간
mkswap   → 스왑 파티션 포맷
swapon   → 스왑 활성화
```

---

---

# ① 메모리 확인

```bash
free -h          # 사람이 읽기 쉬운 단위 (MB, GB)
free -m          # MB 단위
free -g          # GB 단위
free -s 3        # 3초마다 갱신
```

## free 출력 해석

```bash
free -h
#               total    used    free   shared  buff/cache  available
# Mem:           3.8G    1.2G    800M     45M       1.8G       2.4G
# Swap:          2.0G      0B    2.0G
```

```
Mem 행:
  total      = 전체 RAM
  used       = 실제 사용 중
  free       = 완전히 비어있음
  buff/cache = OS 가 캐시로 쓰고 있음 (필요하면 반환 가능)
  available  = 실제로 쓸 수 있는 양 (free + buff/cache 반환분)

Swap 행:
  total = 스왑 전체
  used  = 스왑 사용 중 (0이 이상적)
  free  = 남은 스왑

주의:
  free 가 작아도 available 이 크면 괜찮음
  Swap used 가 크면 메모리 부족 → 성능 저하
```

---

---

# ② 가상 메모리 & 스왑 개념

```
RAM:
  빠름 / 비쌈 / 재부팅 시 초기화
  프로세스가 실제로 사용하는 메모리

Swap:
  디스크 공간을 RAM 처럼 쓰는 가상 메모리
  느림 (디스크라서)
  RAM 이 가득 찼을 때 최후의 보루

OOM (Out Of Memory):
  RAM + Swap 전부 가득 → 프로세스 강제 종료 (OOM Killer)
  DB / 웹서버 같은 중요 프로세스 죽는 대참사
  → Swap 이 있으면 완충지대 역할
```

---

---

# ③ 스왑 파티션 생성 ⭐️

## fdisk 로 스왑 파티션 생성

```bash
sudo fdisk /dev/sdb

# 대화형 명령어
n       # 새 파티션
p       # 주 파티션
2       # 파티션 번호 (1번이 이미 있으면 2번)
        # 시작 섹터 (Enter = 기본값)
+256M   # 크기 지정
t       # 파티션 타입 변경
82      # Linux swap (기본 83=Linux → 82=swap 으로 변경)
p       # 확인
w       # 저장
```

```
파티션 타입:
  83 = Linux (일반 파티션 기본값)
  82 = Linux swap  ← 스왑용으로 변경 필수
```

```bash
# 커널에 갱신
sudo partprobe
```

## mkswap — 스왑 포맷

```bash
sudo mkswap /dev/sdb2
# Setting up swapspace version 1...
# UUID=xxxx-xxxx

# 일반 파일시스템 포맷과 다름:
# mkfs.ext4 → 일반 데이터 저장용
# mkswap    → 스왑 전용 포맷
```

## swapon / swapoff — 스왑 활성화 / 비활성화

```bash
# 스왑 활성화
sudo swapon /dev/sdb2

# 확인
free -h           # Swap 행에 용량 표시됨
swapon -s         # 활성화된 스왑 목록

# 스왑 비활성화
sudo swapoff /dev/sdb2
free -h           # Swap 다시 0 으로
```

## mount vs swapon 차이

```
일반 파티션:
  mkfs.ext4 → mount → 디렉토리로 접근

스왑 파티션:
  mkswap → swapon → 메모리로 사용 (디렉토리 없음)

스왑은 파일시스템이 아니라 메모리 확장 공간
→ mount 안 함 / 디렉토리로 접근 불가
```

---

---

# ④ 메모리 모니터링 명령어

```bash
# 실시간 메모리 통계
vmstat 1              # 1초마다 갱신
vmstat 1 10           # 1초마다 10번

# /proc 에서 직접 확인
cat /proc/meminfo     # 상세 메모리 정보

# 프로세스별 메모리 사용
ps aux --sort=-%mem | head -10   # 메모리 많이 쓰는 프로세스 TOP 10
top                              # 실시간 (M 키로 메모리 정렬)
htop                             # 더 보기 좋은 버전
```

## vmstat 출력 해석

```bash
vmstat 1
# procs  memory        swap   io    system  cpu
# r  b   swpd  free   si  so  bi  bo  in  cs us sy id wa
# 1  0      0  800M    0   0   1   0  10  15  5  1 94  0

# swap:
#   si = swap in  (디스크→메모리 / 클수록 메모리 부족)
#   so = swap out (메모리→디스크 / 클수록 메모리 부족)
#
# si, so 둘 다 0 이면 정상
# 지속적으로 양수면 메모리 증설 필요
```

---

---

# ⑤ 스왑 영구 설정 — /etc/fstab

```bash
# UUID 확인
sudo blkid /dev/sdb2
# UUID="xxxx-xxxx" TYPE="swap"

# /etc/fstab 에 추가
UUID=xxxx-xxxx  none  swap  sw  0  0
#               ↑     ↑     ↑
#          마운트포인트없음  타입  옵션
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|스왑에 `mkfs.ext4` 사용|일반 파티션 포맷 혼동|`mkswap` 사용|
|스왑에 `mount` 사용|디렉토리 연결 시도|`swapon` 사용|
|파티션 타입 안 바꿈|fdisk 에서 82 설정 빠짐|`t → 82` 로 스왑 타입 지정|
|Swap used 높음|메모리 부족|RAM 증설 또는 불필요 프로세스 종료|
|free 가 0 인데 괜찮나|buff/cache 때문|available 확인 (실제 가용량)|