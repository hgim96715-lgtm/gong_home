---
aliases:
  - 커널
  - kernel
  - lsmod
  - modprobe
  - insmod
  - 커널모듈
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Boot_Process]]"
  - "[[Linux_Compile_Install]]"
  - "[[CS_Operating_System]]"
---

# Linux_Kernel — 커널 & 모듈 관리

## 한 줄 요약

```
커널 = 하드웨어와 소프트웨어 사이의 중재자
모듈 = 커널을 재컴파일 없이 기능을 동적으로 확장하는 플러그인
```

---

---

# ① 커널이란

```
커널 (Kernel) = Linux 의 핵심
  하드웨어 직접 제어 (CPU / 메모리 / 디스크 / 네트워크)
  프로세스 / 메모리 / 파일시스템 / 네트워크 관리
  시스템 콜 처리 (프로그램 ↔ 하드웨어 중재)

비유:
  커널 = 공장 관리자
  하드웨어 = 기계
  프로그램 = 작업자
  → 작업자가 기계를 직접 못 만짐
  → 관리자(커널) 통해서만 기계 사용 가능
```

## 커널 버전 확인

```bash
uname -r          # 커널 버전만
# 6.1.0-21-amd64

uname -a          # 전체 시스템 정보
# Linux hostname 6.1.0-21-amd64 #1 SMP Debian 6.1.90

ls /boot/vmlinuz-*    # 설치된 커널 이미지 목록
ls /lib/modules/      # 각 커널별 모듈 디렉토리
```

---

---

# ② 커널 모듈 개념 ⭐️

```
모듈 = 커널 기능을 동적으로 추가/제거할 수 있는 플러그인
      재컴파일 / 재부팅 없이 로드 가능

예시:
  USB 드라이버   → 꽂을 때 자동 로드
  파일시스템     → ext4 / ntfs / vfat
  네트워크 드라이버 → 랜카드 드라이버
  그래픽 드라이버 → nvidia / amdgpu

모듈 파일 확장자: .ko (Kernel Object)
위치: /lib/modules/$(uname -r)/kernel/
```

---

---

# ③ lsmod — 로드된 모듈 목록 ⭐️

```bash
lsmod             # 전체 모듈 목록
lsmod | less      # 페이지 단위로 보기
lsmod | grep 이름 # 특정 모듈 찾기

lsmod | grep joydev
# joydev    32768  0
# ↑ 이름    ↑크기  ↑ Used by (사용 중인 모듈/프로세스 수)
```

## lsmod 출력 해석

```
Module      Size    Used by
joydev      32768   0         ← 아무도 안 씀 (제거 가능)
snd_pcm     131072  2 snd_hda_intel,snd_pcm_oss  ← 2개가 의존
                                                    (제거 불가)

Used by = 0  → 현재 사용 안 함 → rmmod 가능
Used by > 0  → 다른 모듈이 의존 중 → rmmod 실패
```

## 트러블슈팅 패턴 ⭐️

```
새 하드웨어 장착 후 동작 안 할 때:
  1. lsmod | grep 드라이버명   → 모듈 로드됐는지 확인
  2. modinfo 드라이버명        → 모듈 정보 / 의존성 확인
  3. sudo modprobe 드라이버명  → 수동 로드 시도
  4. dmesg | tail -20          → 커널 에러 메시지 확인
```

---

---

# ④ modinfo — 모듈 상세 정보

```bash
modinfo parport
# filename:   /lib/modules/.../parport.ko
# license:    GPL
# description: Parallel-port bus support module
# author:     Philip Blundell
# depends:               ← 비어있으면 의존성 없음
# vermagic:   6.1.0-21-amd64 SMP mod_unload

modinfo joydev
modinfo nvidia    # 그래픽 드라이버 정보
```

```
주요 필드:
  filename   → 모듈 파일 경로
  description → 기능 설명
  depends     → 이 모듈이 필요로 하는 다른 모듈
  author      → 개발자
  license     → GPL / Proprietary
  parm        → 로드 시 전달할 수 있는 파라미터
```

---

---

# ⑤ depmod — 모듈 의존성 갱신

```bash
# 모듈 의존성 데이터베이스 갱신
sudo depmod

# 결과 파일 확인
less /lib/modules/$(uname -r)/modules.dep
```

```
depmod 가 하는 일:
  /lib/modules/$(uname -r)/ 디렉토리의 모듈 분석
  의존성 목록 생성 → modules.dep 파일로 저장

언제 필요한가:
  새 모듈 추가 후
  커널 업데이트 후
  modprobe 가 모듈 못 찾을 때
```

---

---

# ⑥ rmmod — 모듈 제거

```bash
# 제거 전 확인 (Used by = 0 인지 체크)
lsmod | grep joydev
# joydev    32768  0   ← 0 이어야 제거 가능

# 모듈 제거 (성공 시 아무 출력 없음)
sudo rmmod joydev

# 제거 후 확인 (출력 없으면 제거됨)
lsmod | grep joydev
```

```
rmmod 실패하는 경우:
  Used by > 0  → 다른 모듈이 의존 중
  현재 사용 중인 하드웨어가 있음

  → 의존하는 모듈 먼저 제거 후 시도
  → 또는 modprobe -r 사용 (의존성 자동 처리)
```

---

---

# ⑦ insmod — 모듈 수동 로드

```bash
# 전체 경로로 .ko 파일 지정 필수
# $(uname -r) = 현재 커널 버전 자동 치환
sudo insmod /lib/modules/$(uname -r)/kernel/drivers/input/joydev.ko

# 로드 확인 (성공 시 아무 출력 없음)
lsmod | grep joydev
# joydev    32768  0   ← 로드됨
```

```
모듈 표준 위치:
  /lib/modules/$(uname -r)/kernel/
  └── drivers/
       ├── input/     → joydev.ko (조이스틱)
       ├── net/       → 네트워크 드라이버
       ├── gpu/       → 그래픽 드라이버
       └── usb/       → USB 드라이버

$(uname -r) 활용:
  uname -r = 현재 커널 버전 출력
  → 경로에 버전 하드코딩 없이 자동 적용
  예: /lib/modules/6.1.0-21-amd64/kernel/drivers/input/joydev.ko
```

```
insmod 특징:
  .ko 파일의 절대 경로 필요
  의존성 자동 처리 안 함
  → 의존 모듈이 없으면 실패

  주로 개발/테스트 용도:
    직접 컴파일한 .ko 파일 즉시 테스트
    특정 경로의 커스텀 드라이버 로드
```

---

---

# ⑧ modprobe — 의존성 자동 처리 ⭐️

```
insmod 의 업그레이드 버전
의존성 자동 해결 + 이름만으로 로드 가능
실무에서는 modprobe 권장
```

```bash
# 모듈 로드 (의존성 자동 처리)
sudo modprobe joydev
sudo modprobe nvidia

# 모듈 제거 (의존 모듈까지 같이)
sudo modprobe -r joydev

# 드라이 런 (실제 로드 없이 테스트)
sudo modprobe --dry-run joydev

# 설정 보기
cat /etc/modprobe.d/
```

## insmod vs modprobe 비교

|구분|insmod|modprobe|
|---|---|---|
|경로 지정|절대 경로 필수|이름만으로 가능|
|의존성 처리|수동|자동|
|제거|rmmod|modprobe -r|
|용도|개발/테스트|실무 권장|

---

---

# ⑨ 부팅 시 자동 로드 — /etc/modules-load.d/ ⭐️

```
insmod / modprobe 는 임시 로드
→ 재부팅하면 사라짐

영구 설정:
  /etc/modules-load.d/ 에 .conf 파일 추가
  systemd 가 부팅 시 자동으로 읽어서 로드
```

```bash
# 방법 1 — echo + tee 패턴 (권장) ⭐️
echo joydev | sudo tee /etc/modules-load.d/joydev.conf
# 출력: joydev  (화면에도 출력 + 파일에도 저장)

# 확인
cat /etc/modules-load.d/joydev.conf
# joydev

# 방법 2 — 직접 편집
sudo nano /etc/modules-load.d/joydev.conf
# 파일 내용: 모듈 이름 한 줄에 하나씩
# joydev

# 전체 목록 확인
ls /etc/modules-load.d/
```

```
echo | sudo tee 패턴 이유:
  echo joydev > /etc/modules-load.d/joydev.conf
  → ❌ 리다이렉션은 sudo 권한이 파일 쓰기에 안 적용됨

  echo joydev | sudo tee /etc/modules-load.d/joydev.conf
  → ✅ tee 앞에 sudo → 파일 쓰기도 root 권한

systemd 동작:
  부팅 시 /etc/modules-load.d/*.conf 파일을 전부 읽음
  파일 안 모듈 이름을 한 줄씩 modprobe 로 로드
```

## 블랙리스트 — 특정 모듈 차단

```bash
# 특정 모듈이 자동 로드되지 않게 차단
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf

# 적용
sudo depmod
sudo update-initramfs -u   # Debian/Ubuntu
```

```
언제 씀:
  오픈소스 GPU 드라이버(nouveau) 차단 → nvidia 설치 시
  보안상 불필요한 모듈 차단 (블루투스, 플로피 등)
  충돌 모듈 비활성화
```

---

---

# ⑩ 모듈 명령어 한눈에

|명령어|역할|
|---|---|
|`lsmod`|로드된 모듈 목록|
|`lsmod \| grep 이름`|특정 모듈 확인|
|`modinfo 이름`|모듈 상세 정보|
|`sudo depmod`|의존성 DB 갱신|
|`sudo rmmod 이름`|모듈 제거|
|`sudo insmod /경로/모듈.ko`|경로로 모듈 로드|
|`sudo modprobe 이름`|이름으로 모듈 로드 (의존성 자동)|
|`sudo modprobe -r 이름`|의존성까지 같이 제거|
|`echo 이름 \| sudo tee /etc/modules-load.d/이름.conf`|부팅 시 자동 로드|
|`echo "blacklist 이름" \| sudo tee /etc/modprobe.d/blacklist.conf`|자동 로드 차단|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`rmmod` 실패|Used by > 0|의존 모듈 먼저 제거 / `modprobe -r` 사용|
|`insmod` 실패|의존 모듈 없음|의존 모듈 먼저 로드 / `modprobe` 사용|
|재부팅 후 모듈 사라짐|임시 로드|`/etc/modules-load.d/` 에 등록|
|modprobe 모듈 못 찾음|modules.dep 오래됨|`sudo depmod` 실행|
|모듈 자동 로드 차단 실패|blacklist 미설정|`/etc/modprobe.d/blacklist.conf` 추가|

---

---

# ⑪ 커널 컴파일 (리눅스 마스터 1급) ⭐️

## 왜 커널을 컴파일하나

```
패키지 매니저의 커널은 범용적으로 만들어짐
→ 불필요한 기능 포함 / 특정 하드웨어 최적화 불가

직접 컴파일하면:
  필요한 기능만 포함 → 가볍고 빠름
  특정 하드웨어 최적화
  최신 커널 버전 사용
  커스텀 패치 적용

실무에서는 거의 안 함 (임베디드 / 연구 목적)
리눅스 마스터 1급 시험에는 출제됨
```

## 커널 컴파일 전체 흐름

```
1. 소스 다운로드     kernel.org 에서 tar.xz 다운로드
2. 압축 해제         tar -xvf linux-*.tar.xz
3. 설정              make menuconfig (TUI 설정 화면)
4. 컴파일            make -j$(nproc)
5. 모듈 설치         sudo make modules_install
6. 커널 설치         sudo make install
7. GRUB 갱신         sudo grub2-mkconfig -o /boot/grub2/grub.cfg
8. 재부팅 확인       uname -r
```

## 핵심 명령어

```bash
# 1. 빌드 도구 설치
sudo apt-get install build-essential libncurses-dev bison flex libssl-dev

# 2. 현재 커널 설정 복사 (기존 설정 기반으로 시작)
cp /boot/config-$(uname -r) .config
make olddefconfig       # 새 옵션을 기본값으로 자동 설정

# 3. TUI 설정 메뉴 (옵션 추가/제거)
make menuconfig
# 방향키로 탐색 / Space 로 선택
# [*] = 커널에 직접 포함
# [M] = 모듈로 컴파일
# [ ] = 제외

# 4. 컴파일 (시간 오래 걸림 — 수십 분)
make -j$(nproc)

# 5. 모듈 설치
sudo make modules_install
# /lib/modules/커널버전/ 에 설치됨

# 6. 커널 이미지 설치
sudo make install
# /boot/vmlinuz-버전 / /boot/initramfs-버전 생성

# 7. GRUB 갱신
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# 8. 재부팅 후 확인
uname -r   # 새 커널 버전 확인
```

## 설정 옵션 형식

```
make config      → 모든 옵션을 하나씩 묻는 방식 (불편)
make menuconfig  → TUI 메뉴 방식 (권장) ⭐️
make xconfig     → GUI 방식 (X 환경 필요)
make defconfig   → 기본 설정으로 초기화
make oldconfig   → 기존 .config 기반 새 옵션만 물음
```

## 커널 컴파일 vs 모듈 컴파일

```
커널 전체 컴파일:
  make -j$(nproc)
  수십 분 소요
  vmlinuz (커널 이미지) + .ko (모듈) 생성

특정 드라이버만 모듈로 컴파일:
  make M=drivers/input modules
  빠름 (해당 디렉토리만)
  .ko 파일 생성 → insmod 로 즉시 로드
  드라이버 개발 시 활용
```