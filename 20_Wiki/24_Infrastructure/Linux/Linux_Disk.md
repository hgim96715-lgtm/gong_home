---
aliases:
  - 리눅스 디스크
  - fdisst
  - mount
  - fstab
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Filesystem]]"
  - "[[Linux_Basic_Commands]]"
  - "[[Linux_Memory]]"
---

# Linux_Disk — 디스크 & 파일시스템 관리

## 한 줄 요약

```
fdisk  → 파티션 생성
mkfs   → 파일시스템 포맷
mount  → 마운트 (디렉토리에 연결)
fstab  → 영구 마운트 (재부팅 후에도 유지)
```

---

---

# ① 디스크 확인 명령어

```bash
lsblk                    # 디스크 트리 구조로 보기
lsblk -l                 # 리스트 형태
lsblk /dev/sdb           # 특정 디스크만

sudo fdisk -l            # 전체 디스크 상세 정보
sudo fdisk -l /dev/sdb   # 특정 디스크만

df -h                    # 마운트된 파일시스템 사용량
df -h /mnt/data          # 특정 경로만

sudo blkid /dev/sdb1     # UUID / 파일시스템 타입 확인
```

```
lsblk 출력 예시:
  NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda      8:0    0   20G  0 disk
  └─sda1   8:1    0   20G  0 part /
  sdb      8:16   0    2G  0 disk
  └─sdb1   8:17   0  500M  0 part /mnt/data
```

---

---

# ② fdisk — 파티션 생성 ⭐️

```
파티션 = 디스크를 논리적으로 나눈 구역
fdisk 대화형 인터페이스 → w 누르기 전까지 실제 반영 안 됨
실수하면 q 로 저장 없이 종료
```

## 파티션 생성 과정

```bash
sudo fdisk /dev/sdb    # 대화형 모드 진입

# 대화형 명령어
n    # 새 파티션 생성 (new)
p    # 주 파티션 (primary)
1    # 파티션 번호
     # 시작 섹터 (Enter = 기본값)
+500M  # 크기 지정 (500MB)
p    # 현재 상태 확인 (print)
w    # 저장 후 종료 (write)
q    # 저장 없이 종료
```

## 파티션 테이블 갱신

```bash
# 커널이 파티션 인식 못 할 때
sudo partprobe

# 확인
lsblk /dev/sdb
```

---

---

# ③ mkfs — 파일시스템 포맷

```
파티션 생성 = 땅에 금 긋기
포맷(mkfs)  = 그 땅을 사용 가능하게 평탄화

기존 데이터 전부 삭제하는 파괴적 작업
sudo 권한 필요
```

```bash
# ext4 파일시스템으로 포맷
sudo mkfs.ext4 /dev/sdb1

# 포맷 후 확인
sudo blkid /dev/sdb1
# /dev/sdb1: UUID="xxxx-xxxx" TYPE="ext4"

# 파일시스템 메타데이터 확인
sudo dumpe2fs -h /dev/sdb1
```

## 파일시스템 종류

```
ext4     리눅스 표준 / 가장 많이 씀
xfs      대용량 / 고성능 (RHEL 기본)
ntfs     Windows 호환
vfat     USB / SD카드
```

---

---

# ④ mount / umount — 마운트 ⭐️

```
마운트 = 파티션을 디렉토리에 연결
마운트 포인트 = 연결되는 디렉토리
마운트 후 그 디렉토리로 파일 읽기/쓰기 가능

Windows 의 D드라이브 개념과 달리
리눅스는 모든 저장소를 디렉토리 트리로 편입
```

## 마운트

```bash
# 마운트 포인트 생성
sudo mkdir /mnt/data

# 마운트
sudo mount /dev/sdb1 /mnt/data

# 확인
mountpoint /mnt/data    # 마운트 여부
df -h /mnt/data         # 사용량
mount | grep /mnt/data  # 마운트 목록에서 확인

# 파일 생성 테스트
sudo chown labex:labex /mnt/data
touch /mnt/data/testfile
ls -l /mnt/data
# lost+found  testfile
# ↑ lost+found = ext4 표준 / 파일시스템 복구용 디렉토리
```

## 언마운트

```bash
cd ~                    # 마운트 포인트 밖으로 먼저 나오기!
sudo umount /mnt/data

# 확인
mountpoint /mnt/data    # not a mountpoint

# 마운트 포인트 삭제
sudo rmdir /mnt/data
```

```
⚠️ target is busy 에러:
  현재 디렉토리가 마운트 포인트 안에 있으면 발생
  cd ~ 로 빠져나온 뒤 umount 실행

  다른 프로세스가 파일 열고 있어도 발생
  → lsof /mnt/data 로 사용 중인 프로세스 확인
```

---

---

# ⑤ /etc/fstab — 영구 마운트 ⭐️

```
mount 명령 = 임시 (재부팅 시 사라짐)
/etc/fstab = 영구 마운트 설정 파일
재부팅해도 자동으로 마운트됨

⚠️ fstab 오타 → 부팅 시 응급 복구 모드(Emergency Mode) 진입
→ 편집 전 반드시 백업 / 편집 후 mount -a 로 검증
```

## fstab 구조

```
UUID=xxxx  /mnt/data  ext4  defaults  0  2
①          ②          ③     ④       ⑤  ⑥

① 장치      UUID 사용 권장 (/dev/sdb1 은 부팅 순서에 따라 바뀔 수 있음)
② 마운트 포인트
③ 파일시스템 타입
④ 마운트 옵션 (defaults = rw,suid,dev,exec,auto,nouser,async)
⑤ dump 백업 여부 (0=안함)
⑥ fsck 검사 순서 (0=안함 / 1=루트 / 2=나머지)
```

## fstab 설정 순서

```bash
# 1. 백업 (필수!)
sudo cp /etc/fstab /etc/fstab.bak

# 2. UUID 확인
sudo blkid /dev/sdb1
# UUID="a1b2c3d4-..." TYPE="ext4"

# 3. fstab 편집
sudo vim /etc/fstab
# 아래 줄 추가:
# UUID=a1b2c3d4-...  /data  ext4  defaults  0  2

# 4. 마운트 포인트 생성
sudo mkdir /data

# 5. fstab 검증 (재부팅 없이 테스트)
sudo mount -a

# 6. 확인
df -h | grep /data
```

```
UUID 사용 이유:
  /dev/sdb1 은 부팅 순서에 따라 /dev/sdc1 이 될 수 있음
  UUID 는 디스크 고유 식별자 → 항상 동일
```

---

---

# ⑥ 전체 흐름 요약

```
1. lsblk         → 디스크 확인
2. fdisk         → 파티션 생성
3. partprobe     → 커널에 파티션 테이블 갱신 요청
4. mkfs.ext4     → 파일시스템 포맷
5. blkid         → UUID 확인
6. mkdir         → 마운트 포인트 생성
7. mount         → 임시 마운트
8. /etc/fstab    → 영구 마운트 등록
9. mount -a      → fstab 검증
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`umount: target is busy`|마운트 포인트 안에 있음|`cd ~` 로 나온 뒤 umount|
|재부팅 후 마운트 사라짐|mount 만 하고 fstab 안 함|`/etc/fstab` 에 등록|
|부팅 후 Emergency Mode|fstab 오타 / 잘못된 UUID|fstab 편집 후 `mount -a` 검증 필수|
|fdisk 실수로 저장|w 눌러버림|편집 중에는 q 로 종료 / w 신중하게|
|파티션 인식 안 됨|커널 갱신 안 됨|`sudo partprobe` 실행|