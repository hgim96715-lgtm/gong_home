---
aliases:
  - NTP
  - chrony
  - 시간동기화
  - 서버시간
  - timdatectl
tags:
  - Linux
related:
  - "[[System_Uptime]]"
  - "[[Network_Status]]"
  - "[[00_Linux_HomePage]]"
---
## 개념 한 줄 요약

**"현대 리눅스의 시간 관리는 '가벼운 기본 내장(timesyncd)'과 '정밀한 전문가용(chrony)'으로 나뉜다."** 

* **`systemd-timesyncd`:** 최신 우분투(Ubuntu)에 기본 내장된 "가벼운" 클라이언트. 별도 설치 없이 바로 작동함. 
* **`chrony`:** 정밀한 동기화가 필요한 서버(DB, 금융)를 위한 사실상의 **업계 표준(De Facto Standard)**. 
* *(참고: 예전에 쓰던 `ntp` 패키지는 이제 기본 저장소에서 빠지는 추세라 잘 안 씁니다.)*

---

## 왜 필요한가? (Why)

**문제점:**
- "로그 파일을 봤는데, 웹 서버는 10시 00분에 에러가 났고 DB 서버는 10시 02분에 에러가 났대요. 같은 사건 맞나요?" (시간 불일치로 인한 디버깅 불가)
- "OTP(2단계 인증) 로그인이 자꾸 실패해요." (시간이 30초만 틀어져도 인증 코드는 거부됨)

**해결책:**
- 일반 서버는 **`timesyncd`** 가 알아서 시간을 맞춰주니 건들 필요가 없다.
- 초정밀도가 필요한 DB 서버는 **`chrony`** 를 설치해서 오차를 마이크로초(μs) 단위로 줄인다.

---
##  Code Core Points: ① 기본 내장 `systemd-timesyncd`

우분투를 설치하면 이미 돌고 있는 녀석입니다. **`timedatectl`** 명령어로 관리합니다.

```bash
# 1. 현재 시간 및 동기화 상태 확인 (가장 많이 씀)
# "System clock synchronized: yes" 라고 떠야 정상!
timedatectl status

# 2. 강제로 시간 동기화 켜기
sudo timedatectl set-ntp on

# 3. 타임존(Timezone) 한국으로 변경하기
# "서버 시간이 UTC라 불편해요!" -> 이거 한 방이면 해결.
sudo timedatectl set-timezone Asia/Seoul
```

---
## Code Core Points: ② 서버용 표준 `chrony`

AWS, CentOS, RHEL 등 엔터프라이즈 환경의 기본값입니다.
`timesyncd`보다 훨씬 강력하고 빠릅니다.

**특징:**

- **빠른 수렴 (Fast Convergence):** 시간이 틀어졌을 때 NTP보다 훨씬 빨리 정상 궤도로 돌아옵니다.
- **가상화 최적화:** 클라우드(AWS, Google Cloud) 환경에서 더 잘 작동합니다

```bash
# 1. 설치 (우분투에서는 직접 깔아야 함)
# 설치하면 timesyncd는 자동으로 꺼지고 chrony가 지휘봉을 잡음.
sudo apt install chrony

# 2. 상태 확인 (Real-time tracking)
# "지금 시간 오차가 얼마나 나?" (System time : 0.00001 seconds fast/slow)
chronyc tracking

# 3. 동기화 소스 확인
# "누구랑 시간 맞추고 있어?" (* 표시가 현재 대장 서버)
chronyc sources -v

# 4. 실제 시간 확인 
date
```

---
## 상세 분석: 도대체 뭘 써야 해요?

| **구분**    | **systemd-timesyncd** | **chrony**         | **ntp (Legacy)** |
| --------- | --------------------- | ------------------ | ---------------- |
| **포지션**   | **일반 사용자 / 웹 서버**     | **DB / 금융 / 클러스터** | 구형 시스템 유지보수      |
| **설치 여부** | Ubuntu 기본 내장 (설치 X)   | 별도 설치 필요 (`apt`)   | 별도 설치 필요         |
| **정밀도**   | 적당함 (SNTP)            | **매우 정밀함**         | 정밀함              |
| **특징**    | 가볍고 설정이 필요 없음         | 네트워크 끊겨도 오차 보정 강력함 | 설정이 복잡하고 느림      |

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`rdate` 명령어로 시간 맞추면 안 되나요?"

- **절대 금지 (Old School).** 
- 예전 책에는 `rdate -s time.bora.net` 하라고 되어 있는데, 이건 시간을 '확' 바꿔버리는 방식이라 실행 중인 프로그램(DB, 로그)에 치명적입니다. 
- 요즘 리눅스에는 아예 설치도 안 되어 있습니다.

### ② "타임존을 바꿨는데 로그 시간은 그대로 UTC에요."

- `timedatectl set-timezone`으로 시스템 시간은 바꿨지만,
- 실행 중인 프로그램(Java, Python, DB 등)은 **"켜질 때의 시간대"** 를 기억하고 있을 수 있습니다.
- **해결:** 해당 프로그램(서비스)을 재시작(`systemctl restart ...`)해줘야 바뀐 타임존을 인식합니다.

### ③ "`chrony`를 깔았는데 `timedatectl`이 에러가 나요."

- 에러가 아니라 **"NTP service: active"** 라고 뜨면 정상입니다.
- `chrony`가 설치되면 `timesyncd`는 "형님이 오셨군요" 하고 뒤로 빠집니다(Inactive). 둘 중 하나만 돌면 됩니다.

### ④ "Docker 컨테이너에서 `timedatectl`이 안 돼요!" 

- **증상:** `System has not been booted with systemd as init system.` 에러 발생.
- **원인:** `timedatectl`은 **systemd**가 실행 중이어야 쓸 수 있는데, 컨테이너는 가벼움을 위해 **PID 1**이 `systemd`가 아니라 보통 `bash`, `sh`, `tini` 등으로 실행됩니다.
- **해결:** **정상적인 상황(에러 아님)** 입니다. 컨테이너는 독립적인 **OS**가 아니라 호스트와 커널을 공유하는 **격리된 프로세스 환경**이므로, 시간은 호스트 서버의 시간을 그대로 따라갑니다. (호스트 시간을 맞추면 해결!)


---
## [Troubleshooting]

### 트러블슈팅: "506 Cannot talk to daemon"

**의미:** `chronyc`(명령어 치는 애)가 `chronyd`(실제 일하는 애)를 찾을 수 없음. 즉, **chrony가 꺼져 있음.**

#### 상황 1. 일반 리눅스 서버 (Ubuntu, CentOS 등)

대부분 **꺼져 있어서** 그렇습니다. 켜주면 해결됩니다.

```bash
# 1. 상태 확인 (아마 'inactive'나 'failed'라고 뜰 겁니다)
sudo systemctl status chrony

# 2. 서비스 시작 (깨우기)
sudo systemctl start chrony

# 3. 다시 확인
chronyc tracking
```

#### 상황 2. Docker / WSL / 경량 컨테이너 환경 🐳

**`systemctl` 명령어가 안 먹히는 환경**이라서 데몬이 자동으로 안 켜진 겁니다. 
수동으로 데몬을 백그라운드에서 실행시켜야 합니다.

```bash
# 1. 수동으로 데몬 실행 (경로가 다를 수 있으니 확인)
# 보통 /usr/sbin/chronyd 또는 /usr/sbin/chronyd -d (디버그모드)
sudo /usr/sbin/chronyd

# 2. 이제 다시 확인해보세요
chronyc tracking
```

