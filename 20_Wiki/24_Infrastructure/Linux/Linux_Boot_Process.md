---
aliases:
  - 부팅과정
  - GRUB
  - BIOS
  - UEFI
  - systemd
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Runlevel]]"
  - "[[Linux_Kernel]]"
  - "[[Linux_Systemctl]]"
---

# Linux_Boot_Process — 부팅 과정

## 한 줄 요약

```
전원 ON → BIOS/UEFI → GRUB → 커널 → init/systemd → 로그인 화면
```

---

---

# ① 전체 부팅 흐름 ⭐️

```
1. 전원 ON
      ↓
2. BIOS / UEFI      하드웨어 초기화 + 부팅 장치 탐색
      ↓
3. GRUB2            부팅할 OS / 커널 선택 메뉴
      ↓
4. 커널 로딩         Linux Kernel 메모리에 올라감
      ↓
5. initramfs         임시 루트 파일시스템 (드라이버 로드)
      ↓
6. init / systemd    PID 1 프로세스 시작 → 모든 서비스 시작
      ↓
7. 로그인 화면        getty / SSH / 데스크탑
```

---

---

# ② BIOS vs UEFI

## BIOS (Basic Input/Output System)

```
구버전 펌웨어 (1970년대~)
전원 ON 시 가장 먼저 실행되는 프로그램
하드웨어 초기화 → POST (Power-On Self Test) → 부팅 장치 탐색

한계:
  MBR (Master Boot Record) 방식 → 디스크 2TB 제한
  부팅 속도 느림
  보안 기능 없음
```

## UEFI (Unified Extensible Firmware Interface)

```
BIOS 의 현대적 후계자
GPT 파티션 → 디스크 용량 제한 없음
Secure Boot → 서명된 부트로더만 실행
빠른 부팅 (Fast Boot)

/boot/efi 파티션에 부트로더 저장 (EFI System Partition)
```

|구분|BIOS|UEFI|
|---|---|---|
|파티션 방식|MBR (2TB 제한)|GPT (무제한)|
|부팅 속도|느림|빠름|
|보안|없음|Secure Boot|
|인터페이스|텍스트|GUI 지원|
|부트로더 위치|MBR (512바이트)|/boot/efi|

## POST (Power-On Self Test)

```
BIOS/UEFI 가 부팅 시 수행하는 하드웨어 자가 진단
  RAM 테스트
  CPU 동작 확인
  키보드 / 마우스 확인
  디스크 탐색

POST 실패 → 삐 소리 (Beep Code) / 화면 에러
POST 성공 → 부팅 장치 탐색 → GRUB 실행
```

---

---

# ③ GRUB2 — 부트로더 ⭐️

```
GRUB = GRand Unified Bootloader
부팅할 OS / 커널을 선택하는 메뉴
여러 OS 가 설치된 경우 선택할 수 있음
```

## 주요 파일

```
/etc/default/grub         → GRUB 기본 설정 파일 (편집 대상)
/etc/grub.d/              → GRUB 메뉴 생성 스크립트 디렉토리
/boot/grub2/grub.cfg      → grub2-mkconfig 로 생성되는 최종 설정
/boot/efi/                → UEFI 환경의 부트 파일
```

## /etc/default/grub 설정

```bash
sudo cat /etc/default/grub

# 주요 항목:
GRUB_TIMEOUT=5             # 메뉴 대기 시간 (초)
GRUB_DEFAULT=0             # 기본 부팅 항목 (0=첫 번째)
GRUB_CMDLINE_LINUX=""      # 커널 부팅 파라미터
GRUB_DISABLE_SUBMENU=true  # 서브메뉴 비활성화
```

```
GRUB_TIMEOUT:
  숫자  = 대기 후 자동 부팅
  -1   = 사용자 선택할 때까지 무한 대기
  0    = 즉시 부팅 (메뉴 안 보임)

GRUB_DEFAULT:
  0         = 첫 번째 항목 (목록은 0부터)
  saved     = 마지막으로 선택한 항목 기억
  "항목이름" = 특정 항목 이름으로 지정
```

## GRUB 설정 수정 & 적용 ⭐️

```bash
# 1. 설정 파일 편집
sudo nano /etc/default/grub

# 예: TIMEOUT 15초, DEFAULT 첫 번째 항목
# GRUB_TIMEOUT=15
# GRUB_DEFAULT=0

# 2. 변경 후 저장 (nano)
# Ctrl + O → Enter → Ctrl + X

# 3. grub.cfg 재생성 (반드시 실행)
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# UEFI 환경인 경우
sudo grub2-mkconfig -o /boot/efi/EFI/grub/grub.cfg
```

```
⚠️ /etc/default/grub 수정만으로는 반영 안 됨
   반드시 grub2-mkconfig 실행해야 grub.cfg 에 반영됨

grub2-mkconfig 가 하는 일:
  /etc/default/grub 설정 읽기
  /etc/grub.d/ 스크립트 실행
  → 최종 /boot/grub2/grub.cfg 생성
```

## 설정 확인 및 검증

```bash
# 설정 파일 확인
sudo grep -E "(GRUB_TIMEOUT|GRUB_DEFAULT)" /etc/default/grub

# grub.cfg 에 반영됐는지 확인
sudo grep -E "set timeout|set default" /boot/grub2/grub.cfg

# 전체 설정 문법 검사
sudo grub2-script-check /boot/grub2/grub.cfg

# 메뉴 항목 확인
sudo grep "menuentry" /boot/grub2/grub.cfg | head -10

# 백업 (수정 전 필수)
sudo cp /boot/grub2/grub.cfg /boot/grub2/grub.cfg.backup
```

## /etc/grub.d/ 디렉토리 구조

```bash
sudo ls -la /etc/grub.d/

# 00_header    → 기본 GRUB2 설정 (timeout, default)
# 01_users     → 사용자 설정
# 10_linux     → Linux 커널 찾아서 메뉴 항목 추가
# 20_linux_xen → Xen 하이퍼바이저 지원
# 30_os-prober → 다른 OS 탐색 (Windows 등)
# 40_custom    → 사용자 정의 메뉴 항목
```

## GRUB2 복구 명령어

```
부팅 실패 시 GRUB 메뉴에서:
  e = 부팅 항목 임시 편집 (한 번만 적용)
  c = GRUB 명령줄 모드 진입

GRUB 명령줄에서:
  ls               → 파티션 목록 확인
  set root=(hd0,1) → 루트 파티션 지정
  linux /boot/vmlinuz root=/dev/sda1  → 커널 로드
  boot             → 부팅 시작
```

---

---

# ④ 커널 로딩

```
GRUB 이 커널 이미지를 메모리에 올림
/boot/vmlinuz-*    = 커널 이미지 (압축)
/boot/initramfs-*  = 초기 램디스크 이미지
```

## initramfs (초기 램디스크)

```
커널이 실제 루트 파일시스템을 마운트하기 전에
필요한 드라이버 / 모듈을 임시로 제공하는 작은 파일시스템

역할:
  디스크 드라이버 로드
  암호화된 파티션 해제 (LUKS)
  LVM / RAID 초기화
  실제 루트 파티션 마운트

/boot/initramfs-버전.img 파일로 저장됨
```

---

---

# ⑤ init / systemd — PID 1

```
커널이 모든 초기화를 마치면
첫 번째 프로세스(PID 1) 실행
→ 모든 다른 프로세스의 부모

구버전: init (SysV init)
현재:   systemd (RHEL 7+, Ubuntu 15+)
```

## systemd 역할

```
시스템의 모든 서비스를 시작/관리
  네트워크 시작
  로그 시스템 시작 (journald)
  SSH 시작
  데이터베이스 시작
  웹서버 시작
  ...

병렬 시작 → 빠른 부팅
의존성 관리 → A 시작 후 B 시작
```

```bash
# PID 1 확인
ps -p 1
# PID  CMD
#   1  /usr/lib/systemd/systemd

# 부팅 시간 분석
systemd-analyze
systemd-analyze blame   # 서비스별 시작 시간
systemd-analyze plot > boot.svg  # 그래프 생성
```

---

---

# ⑥ 런레벨 / 타겟

```
SysV init:  런레벨 0~6 (숫자)
systemd:    타겟 (target) 으로 대체

기본 타겟 확인 및 변경:
```

```bash
# 현재 타겟 확인
systemctl get-default

# 타겟 변경 (재부팅 없이)
systemctl isolate multi-user.target   # CLI 모드
systemctl isolate graphical.target    # GUI 모드

# 기본 타겟 영구 변경
systemctl set-default multi-user.target
```

|런레벨|systemd 타겟|의미|
|---|---|---|
|0|poweroff.target|시스템 종료|
|1|rescue.target|단일 사용자 모드|
|3|multi-user.target|멀티 유저 (CLI)|
|5|graphical.target|멀티 유저 (GUI)|
|6|reboot.target|재부팅|

---

---

# ⑦ 부팅 문제 해결

## 커널 파라미터 임시 변경

```
GRUB 메뉴에서 e 키 → 부팅 항목 편집
linux 줄 끝에 파라미터 추가:
  single       = 단일 사용자 모드 (복구)
  rd.break     = initramfs 단계에서 멈춤 (루트 비밀번호 변경)
  systemd.unit=rescue.target = 복구 모드
Ctrl+X 로 부팅
```

## 루트 비밀번호 분실 시 복구

```bash
# GRUB 편집 모드에서 linux 줄에 추가:
rd.break

# initramfs 쉘에서:
mount -o remount,rw /sysroot
chroot /sysroot
passwd root          # 새 비밀번호 설정
touch /.autorelabel  # SELinux 레이블 재생성
exit
reboot
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|grub 수정했는데 반영 안 됨|grub2-mkconfig 안 실행|`sudo grub2-mkconfig -o /boot/grub2/grub.cfg`|
|GRUB_TIMEOUT=0 으로 메뉴 안 보임|즉시 부팅 설정|GRUB_TIMEOUT=5 로 변경 후 재적용|
|grub.cfg 직접 편집|재부팅 시 덮어써짐|`/etc/default/grub` 편집 후 mkconfig|
|UEFI 환경에서 경로 다름|BIOS 경로 사용|`/boot/efi/EFI/grub/grub.cfg` 사용|
|부팅 후 환경 달라짐|런레벨/타겟 변경됨|`systemctl set-default` 로 영구 변경|