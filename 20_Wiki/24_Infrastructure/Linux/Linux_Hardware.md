---
aliases:
  - 하드웨어
  - lsblk
  - lspci
  - udevadm
  - /proc
  - /sys
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Disk]]"
  - "[[Linux_Kernel]]"
  - "[[Linux_Memory]]"
---

# Linux_Hardware — 하드웨어 장치 탐색

## 한 줄 요약

```
lsblk   → 블록 장치 (디스크) 목록
lspci   → PCI 버스 장치 목록
udevadm → 장치 속성 상세 조회
/proc   → 커널이 실시간 노출하는 시스템 정보
/sys    → 하드웨어 장치 계층 구조
```

---

---

# ① lsblk — 블록 장치 목록 ⭐️

```
블록 장치 = 데이터를 블록 단위로 읽고 쓰는 저장 장치
  하드 드라이브 / SSD / USB / NVMe

lsblk = List Block devices
  트리 구조로 디스크 → 파티션 관계 시각화
```

```bash
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# vda    252:0    0   20G  0 disk
# ├─vda1 252:1    0  19G  0 part /
# └─vda2 252:2    0   1G  0 part [SWAP]

# 전체 정보 출력
lsblk -a

# 특정 장치만
lsblk /dev/vda
```

## 컬럼 해석

```
NAME        장치명 (vda = 가상 디스크 / sda = SATA / nvme = NVMe)
MAJ:MIN     커널 식별 번호 (Major:Minor)
RM          이동식 여부 (1=이동식 USB 등 / 0=고정)
SIZE        크기
RO          읽기 전용 여부 (1=읽기전용 / 0=읽기쓰기)
TYPE        disk=디스크 / part=파티션 / lvm=LVM
MOUNTPOINTS 마운트 경로 (없으면 미마운트)
```

---

---

# ② lshw — 하드웨어 상세 정보

```bash
# lshw 설치 (기본 설치 안 될 수 있음)
sudo apt-get install -y lshw

# 스토리지 컨트롤러 정보
sudo lshw -class storage

# 파티션 / 논리 볼륨 상세 정보
sudo lshw -class volume

# 전체 하드웨어 목록
sudo lshw

# 간단한 요약
sudo lshw -short
```

```
lsblk vs lshw:
  lsblk  → 빠름 / 디스크 구조 파악에 최적
  lshw   → 상세 스펙 (모델명, 제조사, 시리얼)
```

---

---

# ③ udevadm — 장치 속성 조회 ⭐️

```
udev = 커널 장치 관리자
     /dev 디렉토리의 장치 파일을 동적으로 생성/관리

udevadm = udev 정보 조회 도구
```

## 장치 속성 확인

```bash
# 장치 속성 (모델명, 제조사, 시리얼 등)
udevadm info -q property -n /dev/vda
# ID_MODEL=...
# ID_VENDOR=...
# ID_SERIAL=...

# sysfs 경로 확인
udevadm info -q path -n /dev/vda
# /devices/pci0000:00/0000:00:04.0/virtio1/block/vda

# 전체 계층적 속성 (부모 장치까지)
udevadm info -a -p $(udevadm info -q path -n /dev/vda)
```

```
언제 씀:
  USB 꽂을 때마다 이름이 바뀌는 문제 해결
  → udevadm 으로 고유 시리얼 추출
  → 영구 심볼릭 링크(/dev/my_usb) 설정
  → udev 규칙(rule) 작성 시 필수
```

## udev 규칙 예시

```bash
# /etc/udev/rules.d/99-usb.rules
SUBSYSTEM=="block", ATTRS{serial}=="ABC123", SYMLINK+="my_usb"

# 규칙 적용
sudo udevadm control --reload-rules
```

---

---

# ④ lspci — PCI 장치 목록

```
PCI = Peripheral Component Interconnect
    하드웨어 부품이 연결되는 메인보드 버스

lspci = PCI 장치 목록 출력
  그래픽카드 / 네트워크카드 / 디스크 컨트롤러 등
```

```bash
# 설치
sudo apt-get install -y pciutils

# 기본 목록
lspci

# 트리 구조로 보기
lspci -t

# 트리 + 상세 정보 (드라이버 포함)
lspci -tv | more

# 특정 장치 검색
lspci | grep -i vga      # 그래픽카드
lspci | grep -i network  # 네트워크카드
lspci | grep -i storage  # 스토리지
```

```
트러블슈팅 활용:
  GPU / 고속 랜카드 장착 후 인식 안 될 때
  → lspci 로 버스에 물리적으로 잡혔는지 확인
  → 없으면 하드웨어 문제 / 있으면 드라이버 문제
```

---

---

# ⑤ /proc — 실시간 시스템 정보 ⭐️

```
/proc = 가상 파일 시스템 (디스크에 없음)
커널이 실시간으로 노출하는 시스템 정보
cat 으로 읽으면 현재 상태 조회
```

## 주요 파일

```bash
# CPU 정보
cat /proc/cpuinfo
cat /proc/cpuinfo | grep "model name"

# 메모리 정보
cat /proc/meminfo

# 커널 버전
cat /proc/version

# 실행 중인 프로세스 목록
ls /proc/          # 숫자 디렉토리 = 각 PID

# DMA 채널
cat /proc/dma

# 하드웨어 인터럽트 (IRQ)
cat /proc/interrupts
# CPU0  CPU1  ...  IRQ번호  드라이버명

# I/O 포트 할당
less /proc/ioports

# 마운트된 파일시스템
cat /proc/mounts

# RAID 상태
cat /proc/mdstat

# 네트워크 통계
cat /proc/net/dev
```

## /proc/interrupts 활용

```bash
cat /proc/interrupts
# CPU0  CPU1  ...
#   0: 19    0   XT-PIC   timer
#   1: 10    0   XT-PIC   keyboard
# ...

# 특정 네트워크 카드 IRQ 확인
cat /proc/interrupts | grep eth0
```

```
언제 씀:
  네트워크/디스크 성능 안 나올 때
  → 특정 CPU 코어에 인터럽트 과부하 확인
  → IRQ Affinity 설정으로 분산 (튜닝)
```

---

---

# ⑥ /sys — 하드웨어 계층 구조 ⭐️

```
/sys = sysfs 가상 파일 시스템
/proc 이 프로세스/메모리 중심이라면
/sys  는 하드웨어 장치 계층 구조

lsblk / lspci 같은 도구들이 /sys 에서 데이터를 읽어서 출력함
```

## 주요 디렉토리

```bash
# 블록 장치
ls -l /sys/block
# lrwxrwxrwx vda -> ../devices/pci.../block/vda

ls /sys/block/vda
# dev  queue  size  stat  ...

# 장치 크기 (섹터 단위, 1섹터=512바이트)
cat /sys/block/vda/size
# 41943040  ← 41943040 × 512 = 20GB

# 파티션 정보
ls /sys/block/vda/
# vda1 vda2 ...

cat /sys/block/vda/vda1/size

# 버스 유형별 장치 목록
ls /sys/bus
# cpu  i2c  pci  platform  scsi  usb  virtio  ...

ls -l /sys/bus/virtio/devices
# virtio 기반 가상화 장치 목록

# 네트워크 장치
ls /sys/class/net
# eth0  lo  docker0  ...

cat /sys/class/net/eth0/speed      # 링크 속도
cat /sys/class/net/eth0/operstate  # up/down 상태
```

## /sys 직접 읽기 — 스크립트 활용

```bash
# 자동화 스크립트에서 특정 값 직접 읽기
SIZE=$(cat /sys/block/vda/size)
echo "$((SIZE * 512 / 1024 / 1024 / 1024)) GB"

# lsblk grep/awk 보다 정확하고 빠름
```

```
/sys vs lsblk:
  lsblk → 사람이 읽기 좋음 / grep/awk 필요
  /sys  → 스크립트 자동화 최적 / 파일 하나만 읽으면 됨
```

---

---

# ⑦ systool — SCSI 장치

```bash
# 설치
sudo apt-get install -y sysfsutils

# SCSI 버스 장치 목록
systool -b scsi

# virtio 버스
systool -b virtio
```

---

---

# 트러블슈팅 패턴

## 새 디스크 장착 후 확인

```bash
# 1. 커널이 인식했는지
lsblk

# 2. 상세 정보
sudo lshw -class storage

# 3. udev 속성
udevadm info -q property -n /dev/sdb
```

## 새 GPU/네트워크카드 인식 확인

```bash
# 1. PCI 버스에 잡혔는지
lspci | grep -i vga

# 2. 커널 드라이버 로드됐는지
lspci -tv | grep -i vga
lsmod | grep nvidia   # 또는 amdgpu

# 3. dmesg 에러 확인
dmesg | grep -i error | tail -20
```

---

---

# 명령어 한눈에

|명령어|역할|설치|
|---|---|---|
|`lsblk`|블록 장치 트리|기본 설치|
|`lshw -class storage`|스토리지 상세|`apt install lshw`|
|`udevadm info -q property -n /dev/장치`|장치 속성|기본 설치|
|`lspci`|PCI 장치 목록|`apt install pciutils`|
|`lspci -tv`|PCI 트리 + 드라이버||
|`systool -b scsi`|SCSI 장치|`apt install sysfsutils`|
|`cat /proc/interrupts`|하드웨어 인터럽트|기본|
|`cat /proc/dma`|DMA 채널|기본|
|`less /proc/ioports`|I/O 포트|기본|
|`cat /sys/block/장치/size`|장치 크기 (직접)|기본|