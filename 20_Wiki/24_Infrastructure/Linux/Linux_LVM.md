---
aliases:
  - LVM
  - 논리 볼륨
  - pvcreate
  - vgcreate
  - lvcreate
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Disk]]"
  - "[[Linux_RAID]]"
---

# Linux_LVM — 논리 볼륨 관리

## 한 줄 요약

```
LVM = 디스크를 마음대로 늘리고 줄일 수 있는 유연한 스토리지 관리 시스템
기존 파티션: 크기 고정 (변경 어려움)
LVM:        운영 중에도 크기 조정 가능
```

---

---

# ① LVM 이 왜 필요한가

```
기존 방식의 문제:
  /dev/sdb1  → 500GB 파티션 생성
  → 나중에 용량 부족해도 늘리기 매우 어려움
  → 파티션 삭제 후 새로 만들어야 함 → 데이터 날아감

LVM 의 해결:
  서버 운영 중에도 명령어 한 줄로 용량 확장
  디스크 여러 개를 하나처럼 합치거나 나눔
  스냅샷 (특정 시점 백업) 가능
```

---

---

# ② LVM 3단계 구조 ⭐️

```
물리 디스크 → PV (물리 볼륨) → VG (볼륨 그룹) → LV (논리 볼륨)

비유로 이해:
  PV = 쌀 포대 (실제 디스크)
  VG = 창고 (포대를 전부 쌓아둔 공간)
  LV = 통 (창고에서 덜어낸 공간)
```

## PV — Physical Volume (물리 볼륨)

```
실제 디스크 / 파티션을 LVM 이 관리할 수 있게 등록한 것
"이 디스크를 LVM 용으로 쓸게" 라고 선언
```

## VG — Volume Group (볼륨 그룹)

```
PV 여러 개를 하나의 거대한 스토리지 풀로 묶은 것
500GB 디스크 2개 → VG = 1TB 풀
여기서 LV 를 꺼내 씀
```

## LV — Logical Volume (논리 볼륨)

```
VG 에서 원하는 크기만큼 잘라낸 "가상 파티션"
이 위에 파일시스템(ext4) 만들고 마운트해서 사용
운영 중에도 크기 조정 가능 ← LVM 최대 장점
```

## 전체 흐름 한눈에

```
실제 디스크:   /dev/loop20  /dev/loop21
                   ↓              ↓
PV 등록:       pvcreate
                   ↓
VG 생성:       vgcreate labvg    (하나의 풀로 합침)
                   ↓
LV 생성:       lvcreate -L 200M -n lablvm labvg
                   ↓
파일시스템:    mkfs.ext4 /dev/labvg/lablvm
                   ↓
마운트:        mount /dev/labvg/lablvm /lablvm
```

---

---

# ③ 설치 및 가상 디스크 준비

```bash
# LVM + RAID 도구 설치
sudo apt-get update && sudo apt-get install -y lvm2 mdadm

# 가상 디스크 파일 생성 (실제 디스크 없을 때 테스트용)
truncate -s 256M disk1.img disk2.img
ls -lh disk1.img  # 256M 파일 생성됨

# 파일을 블록 장치처럼 연결 (loop 디바이스)
sudo losetup /dev/loop20 disk1.img
sudo losetup /dev/loop21 disk2.img
```

```
truncate = 지정한 크기의 빈 파일 생성
losetup  = 파일을 /dev/loop20 같은 블록 장치로 연결
           마치 실제 디스크처럼 사용 가능
```

---

---

# ④ PV 생성 — pvcreate ⭐️

```bash
# 물리 볼륨으로 등록
sudo pvcreate /dev/loop20 /dev/loop21

# 확인
sudo pvs           # 간단히
sudo pvdisplay     # 상세히
```

```
pvs 출력 예시:
  PV          VG    Fmt  Attr PSize   PFree
  /dev/loop20       lvm2 ---  256.00m 256.00m
  /dev/loop21       lvm2 ---  256.00m 256.00m
```

---

---

# ⑤ VG 생성 — vgcreate ⭐️

```bash
# PV 들을 묶어서 볼륨 그룹 생성
sudo vgcreate labvg /dev/loop20 /dev/loop21

# 확인
sudo vgs           # 간단히
sudo vgdisplay     # 상세히
```

```
vgs 출력 예시:
  VG    #PV #LV #SN Attr  VSize   VFree
  labvg   2   0   0 wz--n 504.00m 504.00m
  ↑ 두 디스크(256M x 2) 합쳐서 ~504MB 풀 생성
```

---

---

# ⑥ LV 생성 → 포맷 → 마운트 ⭐️

```bash
# LV 생성 (VG 에서 200MB 할당)
sudo lvcreate -L 200M -n lablvm labvg
#              ↑크기   ↑이름    ↑VG이름

sudo lvs   # LV 목록 확인

# 파일시스템 포맷
sudo mkfs.ext4 /dev/labvg/lablvm

# 마운트 포인트 생성 후 마운트
sudo mkdir /lablvm
sudo mount /dev/labvg/lablvm /lablvm

# 확인
df -h /lablvm
```

```
lvcreate 옵션:
  -L 200M  → 200MB 크기
  -L +100M → 현재보다 100MB 추가
  -n 이름  → LV 이름 지정
```

---

---

# ⑦ 용량 확장 — lvresize ⭐️ (LVM 최대 장점)

```bash
# 200M → 300M 으로 확장
sudo lvresize -r -L 300M /dev/labvg/lablvm
df -h /lablvm   # 300M 으로 늘어남

# 300M → 400M 으로 추가 확장
sudo lvresize -r -L 400M /dev/labvg/lablvm
df -h /lablvm
```

```
-r 옵션이 핵심:
  -r 없으면: LV 껍데기만 커지고 파일시스템은 그대로
            → 실제 사용 공간 안 늘어남
  -r 있으면: LV + 파일시스템 동시 확장 ✅

  무조건 -r 붙이기
```

---

---

# ⑧ 영구 마운트 — /etc/fstab

```bash
# /etc/fstab 에 LVM 마운트 추가
echo '/dev/labvg/lablvm /lablvm ext4 defaults 0 0' | sudo tee -a /etc/fstab

# 검증 (재부팅 없이)
sudo umount /lablvm
sudo mount -a
df -h /lablvm   # 다시 마운트됨
```

---

---

# 명령어 한눈에

|단계|명령어|역할|
|---|---|---|
|PV 등록|`pvcreate /dev/sdb`|디스크를 LVM 에 등록|
|PV 확인|`pvs` / `pvdisplay`|PV 목록|
|VG 생성|`vgcreate VG이름 PV1 PV2`|PV 묶어서 풀 생성|
|VG 확인|`vgs` / `vgdisplay`|VG 목록|
|LV 생성|`lvcreate -L 크기 -n 이름 VG이름`|가상 파티션 생성|
|LV 확인|`lvs` / `lvdisplay`|LV 목록|
|LV 확장|`lvresize -r -L 크기 /dev/VG/LV`|운영 중 크기 조정|
|포맷|`mkfs.ext4 /dev/VG/LV`|파일시스템 생성|
|마운트|`mount /dev/VG/LV /경로`|사용 가능하게 연결|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|lvresize 후 용량 그대로|`-r` 옵션 빠짐|`lvresize -r` 필수|
|LV 만들었는데 쓸 수 없음|mkfs 안 함|`mkfs.ext4` 후 mount|
|재부팅 후 마운트 사라짐|fstab 등록 안 함|`/etc/fstab` 에 추가|
|pvcreate 실패|이미 다른 용도|`wipefs -a /dev/장치` 후 재시도|