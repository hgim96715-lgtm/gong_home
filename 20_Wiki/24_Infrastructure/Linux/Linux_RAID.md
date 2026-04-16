---
aliases:
  - RAID
  - mdadm
  - 미러링
  - 스트라이핑
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Disk]]"
  - "[[Linux_LVM]]"
---

# Linux_RAID — RAID 스토리지 구성

## 한 줄 요약

```
RAID = 디스크 여러 개를 하나처럼 묶어서
      속도를 높이거나 (스트라이핑)
      안전성을 높이는 (미러링) 기술
```

---

---

# ① RAID 가 왜 필요한가

```
문제:
  디스크 1개 → 고장나면 데이터 전부 날아감
  서버 운영 중 디스크 교체 불가 → 서비스 중단

해결:
  RAID = 디스크 여러 개를 조합
  → 하나 고장나도 데이터 보존 (미러링)
  → 여러 개 동시에 써서 속도 향상 (스트라이핑)

LVM vs RAID 차이:
  LVM  → 유연성 (운영 중 크기 조정)
  RAID → 안정성 / 속도 (데이터 보호)
  둘 다 같이 쓰는 경우도 많음
```

---

---

# ② RAID 종류 ⭐️

## RAID 0 — 스트라이핑 (속도↑ / 안전성✗)

```
디스크 2개에 데이터를 반반 나눠서 동시에 씀
→ 속도 2배
→ 하나라도 고장나면 전부 날아감

비유:
  책을 찢어서 짝수 페이지 / 홀수 페이지 두 곳에 나눠 보관
  하나 잃으면 책 못 읽음
  하지만 두 명이 동시에 읽으면 2배 빠름
```

```
디스크 A: [1] [3] [5] [7]   ← 짝수 블록
디스크 B: [2] [4] [6] [8]   ← 홀수 블록
→ 동시에 읽기/쓰기 → 속도 2배
→ A 고장 → [1][3][5][7] 없어짐 → 전체 데이터 손실
```

## RAID 1 — 미러링 (안전성↑ / 용량 반)

```
디스크 2개에 완전히 똑같은 데이터를 동시에 씀
→ 하나 고장나도 다른 하나로 계속 운영
→ 용량은 2개 중 1개만 실제 사용

비유:
  같은 내용을 두 권의 책에 동시에 기록
  하나 잃어도 다른 하나로 읽을 수 있음
  단, 책 2권 써서 1권 분량만 저장
```

```
디스크 A: [1] [2] [3] [4]   ← 원본
디스크 B: [1] [2] [3] [4]   ← 완전 동일 복사본
→ A 고장 → B 로 계속 운영
→ 용량: 256MB x 2 = 256MB (절반만 사용)
```

## RAID 5 — 패리티 (안전성 + 효율)

```
디스크 3개 이상
데이터 + 패리티(오류 복구 정보) 분산 저장
→ 1개 고장나도 복구 가능
→ 용량: (디스크 수 - 1) x 1개 용량

비유:
  책 3권 중 2권에 내용, 1권에 복구코드 저장
  (복구코드도 돌아가면서 다른 디스크에)
  1권 잃어도 나머지 2권으로 복구
```

## RAID 6 — 이중 패리티

```
디스크 4개 이상
패리티 2개 → 2개 동시 고장 복구 가능
대용량 서버에서 사용
```

## RAID 종류 비교표

|구분|최소 디스크|속도|안전성|실제 용량|용도|
|---|---|---|---|---|---|
|RAID 0|2개|↑↑ 빠름|✗ 없음|100%|속도 우선 (동영상 편집)|
|RAID 1|2개|보통|✅ 1개 고장 OK|50%|안전성 우선 (DB)|
|RAID 5|3개|보통|✅ 1개 고장 OK|(n-1)/n|균형 (서버)|
|RAID 6|4개|약간 느림|✅ 2개 고장 OK|(n-2)/n|고안전성|

---

---

# ③ mdadm — 소프트웨어 RAID

```
mdadm = Multiple Disk ADMin
Linux 에서 소프트웨어 RAID 구성 도구
하드웨어 RAID 카드 없이 OS 에서 RAID 구현
```

## 설치

```bash
sudo apt-get install -y mdadm
```

## 가상 디스크 준비 (테스트 환경)

```bash
truncate -s 256M disk3.img disk4.img
sudo losetup /dev/loop22 disk3.img
sudo losetup /dev/loop23 disk4.img
```

---

---

# ④ RAID 1 구성 실전 ⭐️

## RAID 어레이 생성

```bash
sudo mdadm --create /dev/md0 \
    --level=1 \
    --raid-disks=2 \
    /dev/loop22 /dev/loop23
# 확인 메시지 나오면 'y' 입력
```

```
--create /dev/md0   → RAID 장치 이름 (md0 = Multiple Device 0)
--level=1           → RAID 1 (미러링)
--raid-disks=2      → 디스크 2개 사용
```

## RAID 상태 확인

```bash
cat /proc/mdstat
# md0 : active raid1 loop23[1] loop22[0]
#       261120 blocks super 1.2 [2/2] [UU]
#               ↑ [UU] = 두 디스크 정상 (U=Up / _=Down)
```

```
/proc/mdstat 해석:
  [UU]   = 2개 모두 정상
  [U_]   = 1개 고장 (언더바 = 고장)
  resync = 동기화 진행 중 (새로 추가 시)
  rebuild = 교체 후 복구 중
```

## 포맷 → 마운트

```bash
sudo mkfs.ext4 /dev/md0

sudo mkdir /labraid
sudo mount /dev/md0 /labraid

df -h /labraid
# /dev/md0  255M  ...  /labraid
# 2개(256M x 2) 중 1개 용량만 사용 가능 ← RAID 1 특성
```

---

---

# ⑤ RAID 영구 설정 ⭐️

## RAID 설정 저장 — mdadm.conf

```bash
# RAID 정보 확인
sudo mdadm --detail --scan

# mdadm.conf 에 저장 (재부팅 후에도 RAID 인식)
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf

cat /etc/mdadm/mdadm.conf   # 확인
```

## tee 명령어 ⭐️

```
파이프(|) 로 연결할 때 쓰는 명령어
→ 출력을 화면 + 파일 두 곳에 동시에 보냄

명령어 | tee 파일명        # 파일 덮어쓰기
명령어 | tee -a 파일명     # 파일에 추가 (-a = append)
```

```bash
# tee 없이 >> 로 쓰면 안 되나?
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
# ❌ >> 는 sudo 권한이 파일 열기에 안 적용됨
#    명령어에만 sudo / 파일 쓰기는 일반 권한으로 → Permission denied

sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
# ✅ tee 앞에 sudo → 파일 쓰기도 root 권한으로 실행
```

```
tee vs >> 차이:
  >>          : 리다이렉션 (파일에만 저장 / 화면 출력 없음)
  tee         : 화면 + 파일 동시 (내용 확인하면서 저장)
  tee -a      : 화면 + 파일에 추가 (기존 내용 유지)
  sudo tee -a : root 권한으로 파일에 추가

실무 패턴:
  sudo 명령어 결과를 시스템 파일에 저장할 때
  → 항상 | sudo tee -a 파일명 패턴 사용
```

```bash
# 다른 예시들
echo "설정값" | sudo tee -a /etc/hosts        # hosts 에 추가
echo "마운트줄" | sudo tee -a /etc/fstab      # fstab 에 추가
cat 설정파일 | sudo tee /etc/nginx/nginx.conf  # nginx 설정 덮어쓰기
```

## fstab 에 마운트 등록

```bash
# LVM + RAID 둘 다 fstab 등록
echo '/dev/labvg/lablvm /lablvm ext4 defaults 0 0' | sudo tee -a /etc/fstab
echo '/dev/md0 /labraid ext4 defaults 0 0' | sudo tee -a /etc/fstab

tail -n 2 /etc/fstab   # 마지막 2줄 확인
```

## 검증 — 재부팅 없이 테스트

```bash
sudo umount /lablvm
sudo umount /labraid
sudo mount -a          # fstab 전체 마운트
df -h /lablvm /labraid # 둘 다 마운트됨 확인
```

```
fstab 설정 후 재부팅 전 반드시:
  umount → mount -a 로 검증
  오타 있으면 Kernel Panic 으로 부팅 멈춤
```

---

---

# ⑥ LVM vs RAID 언제 쓰나

```
RAID 1:
  "디스크 하나 고장나도 서비스 계속"
  → DB 서버 / 중요 데이터
  → 안전성 최우선

LVM:
  "나중에 용량 늘려야 할 것 같다"
  → 운영 중 확장 필요한 환경
  → 유연성 우선

LVM + RAID 같이:
  RAID 로 안전성 확보 → 그 위에 LVM 구성
  → 안전하면서 유연한 스토리지
  → 실무 서버에서 많이 쓰는 구성
```

---

---

# 명령어 한눈에

|명령어|역할|
|---|---|
|`mdadm --create /dev/md0 --level=1 --raid-disks=2 디스크1 디스크2`|RAID 생성|
|`cat /proc/mdstat`|RAID 상태 확인|
|`mdadm --detail /dev/md0`|RAID 상세 정보|
|`mdadm --detail --scan`|RAID 설정 출력|
|`mdadm --detail --scan \| tee -a /etc/mdadm/mdadm.conf`|영구 저장|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|재부팅 후 RAID 사라짐|mdadm.conf 미저장|`--detail --scan >> mdadm.conf`|
|RAID 1 인데 용량 절반|정상 동작|미러링 특성 — 1개 용량만 사용|
|[U_] 상태|디스크 1개 고장|`mdadm /dev/md0 --add 새디스크` 로 교체|
|mount -a 후 에러|fstab 오타|fstab 다시 확인 후 mount -a|