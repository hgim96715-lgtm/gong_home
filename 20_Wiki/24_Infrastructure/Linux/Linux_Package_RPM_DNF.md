---
aliases:
  - rpm
  - yum
  - dnf
  - 패키지관리
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Package_APT]]"
  - "[[Linux_Basic_Commands]]"
---
# Linux_Package_RPM_DNF — RPM / YUM / DNF 패키지 관리

## 한 줄 요약

```
RPM  = 패키지 파일 자체 (.rpm 파일 직접 다루기)
YUM  = RPM 위에서 의존성 자동 처리 (구버전)
DNF  = YUM 의 후계자 (RHEL 8+ / Fedora 기본)

Ubuntu/Debian 계열 → apt
Red Hat/CentOS/Fedora 계열 → rpm/yum/dnf
```

## 배포판별 패키지 관리자

```
Debian 계열:  apt / apt-get / dpkg
  Ubuntu / Debian / Linux Mint

Red Hat 계열: rpm / yum / dnf
  RHEL / CentOS / Fedora / Rocky Linux / AlmaLinux
```

---

---

# ① rpm 기본 개념

```
rpm = Red Hat Package Manager
.rpm 파일 = 소프트웨어를 설치하기 위한 압축 패키지

패키지 파일명 구조:
  bash-5.1.8-6.el9.x86_64.rpm
  ↑    ↑     ↑    ↑    ↑
  이름  버전  릴리즈  OS  아키텍처

rpm 의 한계:
  의존성을 자동으로 처리 못함
  A 를 설치하려면 B 가 필요한데 → 직접 설치해야 함
  → 이 문제를 해결한 게 yum / dnf
```

---

---

# ② rpm 조회 명령어 ⭐️

## 기본 조회 (-q)

```bash
rpm -q bash              # bash 설치됐는지 확인
# bash-5.1.8-6.el9.x86_64  (설치됨)
# package bash is not installed  (미설치)

rpm -qa                  # 설치된 모든 패키지 목록
rpm -qa | grep python    # 이름에 python 포함된 패키지
rpm -qa | wc -l          # 설치된 패키지 총 개수
```

## -qi — 패키지 상세 정보

```bash
rpm -qi bash
# Name        : bash
# Version     : 5.1.8
# Release     : 6.el9
# Architecture: x86_64
# Install Date: Mon 14 Apr 2025
# Size        : 7,692,288
# Summary     : GNU Bourne Again shell
# Description : bash 는 ...
```

```
언제 씀:
  이 패키지 버전이 몇인지 확인
  언제 설치됐는지 확인
  패키지 크기 확인
```

## -qf — 파일 → 패키지 역추적 ⭐️

```bash
# 이 파일이 어느 패키지에서 설치됐나?
rpm -qf /usr/bin/bash
# bash-5.1.8-6.el9.x86_64

rpm -qf /usr/bin/python3
# python3-3.9.18-3.el9.x86_64

rpm -qf /etc/passwd
# setup-2.13.7-10.el9.noarch
```

```
언제 씀:
  출처 모르는 파일 발견 → 어느 패키지에서 왔는지
  특정 실행파일이 어떤 패키지인지
  트러블슈팅 시 파일 → 패키지 역추적
```

## -ql — 패키지가 설치한 파일 목록

```bash
rpm -ql bash
# /usr/bin/bash
# /usr/bin/sh
# /usr/share/man/man1/bash.1.gz
# ...
```

## -qR — 의존성 확인 (이 패키지가 필요로 하는 것)

```bash
rpm -qR bash
# /bin/sh
# glibc >= 2.15
# libreadline.so.8
# ...
```

```
의존성 = 이 패키지가 동작하려면 필요한 다른 패키지/라이브러리
bash 를 실행하려면 glibc 가 있어야 함
```

## -q --whatrequires — 역의존성 (이 패키지를 필요로 하는 것)

```bash
# glibc 를 필요로 하는 패키지들
rpm -q --whatrequires glibc
# bash-5.1.8-6.el9.x86_64
# coreutils-8.32-34.el9.x86_64
# ... (매우 많음)

# 아무도 필요로 하지 않는 패키지 → 안전하게 삭제 가능
rpm -q --whatrequires lsscsi
# no package requires lsscsi
```

## -qc — 설정 파일 목록

```bash
rpm -qc bash
# /etc/skel/.bash_logout
# /etc/skel/.bash_profile
# /etc/skel/.bashrc

# 특정 파일 → 패키지 → 설정파일 목록
rpm -qcf /usr/bin/passwd
# /etc/pam.d/passwd
```

```
언제 씀:
  이 패키지의 설정 파일이 어디 있는지
  패키지 설정 변경 전 파일 위치 파악
```

---

---

# ③ rpm -V — 무결성 검증 ⭐️

```
rpm -V = Verify (검증)
패키지의 파일들이 설치 당시 원본 상태를 유지하는지 검사
해킹 / 변조 / 실수로 설정 변경됐는지 확인
```

## 기본 사용

```bash
rpm -V bash
# (아무 출력 없음) = 모든 파일 원본과 동일 ✅

# 설정 파일 수정 후 검증
echo "labex" | sudo tee -a /etc/at.deny
sudo rpm -V at
# S.5....T. c /etc/at.deny
```

## 출력 플래그 해석 ⭐️

```
S.5....T. c /etc/at.deny
↑↑↑↑↑↑↑↑↑ ↑ ↑
│││││││││   │ └── 파일 경로
│││││││││   └──── c = 설정 파일 (configuration)
│││││││││
│││││││││  각 자리 의미 (. = 정상 / 변경되면 글자 표시):
│        └── . = 마지막 체크 (SELinux 컨텍스트)
│ .          = 변경 없음
S           = Size 파일 크기 변경
 .          = 모드 변경 없음
  5         = MD5 체크섬 변경 (내용 변경)
   .        = 심볼릭 링크 변경 없음
    .       = 소유자 변경 없음
     .      = 그룹 변경 없음
      .     = mtime 변경 없음 → T 이면 시간 변경
       T    = 수정 시간(mTime) 변경
        .   = capability 변경 없음
```

```
자주 보는 패턴:
  S.5....T. = 파일 크기 + 내용 + 시간 변경 → 수정됨
  .......   = 모두 정상

c (소문자):
  c 가 붙으면 설정 파일 (configuration file)
  설정 파일은 의도적으로 수정하는 경우 많음

보안 활용:
  시스템 핵심 바이너리가 변조됐는지 검사
  다른 관리자가 설정 파일 건드렸는지 감사(Audit)
```

```bash
# 특정 파일 소속 패키지 전체 검증
sudo rpm -qVf /etc/at.deny
```

---

---

# ④ rpm 설치 / 삭제

## 설치

```bash
# .rpm 파일 직접 설치
sudo rpm -ivh 패키지.rpm
#             ↑↑↑
#             i = install
#             v = verbose (설치 과정 출력)
#             h = hash (진행률 # 표시)

# 업그레이드 (없으면 설치, 있으면 업그레이드)
sudo rpm -Uvh 패키지.rpm

# 이미 설치된 것 강제 재설치
sudo rpm -ivh --replacepkgs 패키지.rpm
```

## 삭제 — erase

```bash
# 삭제 (단, 다른 패키지가 의존하면 실패)
sudo rpm -e bash

# 삭제 시뮬레이션 — 실제 삭제 안 하고 테스트 ⭐️
sudo rpm -e --test glibc
# error: Failed dependencies:
#   glibc >= 2.15 is needed by bash-...
#   (많은 패키지가 glibc 필요 → 삭제 차단)

sudo rpm -e --test lsscsi
# (아무 에러 없음) = 안전하게 삭제 가능
```

```
--test 가 중요한 이유:
  운영 서버에서 패키지 정리 시
  삭제했더니 다른 중요 서비스가 멈추는 대참사 방지
  반드시 --test 먼저 실행 → 에러 없으면 삭제
```

---

---

# ⑤ rpm2cpio — RPM 내부 파일 추출 ⭐️

```
설치 없이 RPM 파일 안의 내용 확인 / 파일 추출
특정 파일 하나만 손상됐을 때 복구에 활용
```

```bash
# RPM 파일 다운로드 (설치는 안 함)
sudo dnf download bash
ls bash-*.rpm

# RPM 내부 파일 목록 확인 (설치 안 함)
rpm2cpio bash-*.rpm | cpio -t
# ./usr/bin/bash
# ./usr/bin/sh
# ./usr/share/man/...

# RPM 에서 특정 파일 하나만 추출 (복구 시)
rpm2cpio bash-*.rpm | cpio -idv ./usr/bin/bash
# 현재 디렉토리에 ./usr/bin/bash 추출됨
```

```
언제 씀:
  패키지 전체 재설치하면 설정이 날아갈 위험
  → 손상된 파일 하나만 RPM 에서 꺼내서 복구

cpio 옵션:
  -t  = 목록만 보기 (Table of contents)
  -i  = 추출 (extract)
  -d  = 필요한 디렉토리 자동 생성
  -v  = verbose (과정 출력)
```

---

---

# ⑥ YUM / DNF — 의존성 자동 처리 ⭐️

```
rpm 의 한계: 의존성 수동 처리
yum / dnf: 의존성 자동 해결 + 저장소(repository)에서 자동 다운로드

yum  → CentOS 7 이하 / RHEL 7 이하 (구버전)
dnf  → RHEL 8+ / Fedora / Rocky Linux (최신)
     → yum 명령어도 dnf 로 alias 되어 있어 대부분 호환
```

## 기본 명령어

```bash
# 패키지 설치
sudo dnf install 패키지명
sudo dnf install -y 패키지명    # -y 로 확인 없이 자동 설치

# 패키지 제거
sudo dnf remove 패키지명

# 패키지 업데이트
sudo dnf update 패키지명
sudo dnf update                 # 전체 업데이트

# 패키지 검색
dnf search 키워드

# 패키지 정보 확인
dnf info 패키지명

# 설치된 패키지 목록
dnf list installed
dnf list installed | grep python

# 저장소 목록
dnf repolist
```

## dnf vs apt 비교

|기능|dnf (Red Hat 계열)|apt (Debian 계열)|
|---|---|---|
|설치|`dnf install`|`apt install`|
|제거|`dnf remove`|`apt remove`|
|업데이트|`dnf update`|`apt update && apt upgrade`|
|검색|`dnf search`|`apt search`|
|목록|`dnf list installed`|`dpkg -l`|
|저장소 갱신|자동|`apt update` 별도 필요|

---

---

# ⑦ spec 파일 — RPM 패키지 직접 만들기 (리눅스 마스터 1급)

## spec 파일이 뭔가

```
지금까지 배운 건: 이미 만들어진 .rpm 파일을 설치/조회/삭제
spec 파일은:     내가 직접 .rpm 패키지를 만들 때 필요한 설계도

비유:
  .rpm 파일  = 완성된 제품
  spec 파일  = 제품 설계도 (무엇을, 어떻게 만들지 명세)

실무 활용:
  사내 소프트웨어를 RPM 으로 배포할 때
  커스텀 패키지 빌드할 때
  → 데이터 엔지니어 실무에서는 거의 안 씀
  → 리눅스 마스터 1급 시험에는 출제됨
```

## spec 파일 구조

```spec
# myapp.spec

# ─── 헤더 (패키지 기본 정보) ───────────────────
Name:       myapp             # 패키지 이름
Version:    1.0               # 버전
Release:    1%{?dist}         # 릴리즈 번호
Summary:    My application    # 한 줄 요약
License:    GPL               # 라이선스
URL:        https://example.com
Source0:    %{name}-%{version}.tar.gz  # 소스 파일

# ─── 의존성 ────────────────────────────────────
Requires:   bash >= 4.0       # 설치 시 필요한 패키지
BuildRequires: gcc            # 빌드 시 필요한 패키지

# ─── 설명 ──────────────────────────────────────
%description
이 패키지는 myapp 애플리케이션입니다.
자세한 설명을 여기에 작성합니다.

# ─── 빌드 준비 (압축 해제) ─────────────────────
%prep
%autosetup                    # Source0 압축 해제 + 패치 적용

# ─── 빌드 (컴파일) ─────────────────────────────
%build
make %{?_smp_mflags}          # make 로 컴파일

# ─── 설치 (파일 배치) ──────────────────────────
%install
make install DESTDIR=%{buildroot}   # 임시 루트에 설치

# ─── 포함 파일 목록 ────────────────────────────
%files
/usr/bin/myapp               # 실행 파일
/etc/myapp/myapp.conf        # 설정 파일
%doc README.md               # 문서 파일

# ─── 설치 전/후 스크립트 ───────────────────────
%pre
echo "설치 전 실행"

%post
systemctl enable myapp       # 설치 후 서비스 등록

%preun
systemctl stop myapp         # 삭제 전 서비스 중지

%postun
echo "삭제 후 실행"

# ─── 변경 이력 ─────────────────────────────────
%changelog
* Mon Apr 19 2026 labex <labex@example.com> - 1.0-1
- 최초 릴리즈
```

## 섹션 역할 한눈에

|섹션|역할|
|---|---|
|헤더|패키지 이름/버전/설명 정의|
|`%description`|상세 설명|
|`%prep`|소스 압축 해제 / 패치 적용|
|`%build`|컴파일 (make)|
|`%install`|파일을 임시 루트에 배치|
|`%files`|RPM 에 포함할 파일 목록|
|`%pre` / `%post`|설치 전/후 스크립트|
|`%preun` / `%postun`|삭제 전/후 스크립트|
|`%changelog`|버전별 변경 이력|

## spec 파일 → RPM 빌드

```bash
# 빌드 도구 설치
sudo dnf install rpm-build rpmdevtools

# 빌드 디렉토리 구조 생성
rpmdev-setuptree
# ~/rpmbuild/
# ├── BUILD/    ← 빌드 작업 공간
# ├── RPMS/     ← 완성된 .rpm 파일
# ├── SOURCES/  ← 소스 파일 (.tar.gz)
# ├── SPECS/    ← spec 파일 위치
# └── SRPMS/    ← 소스 RPM

# spec 파일로 RPM 빌드
rpmbuild -ba ~/rpmbuild/SPECS/myapp.spec
#         ↑
#         b = build
#         a = all (바이너리 + 소스 RPM 모두)

# 빌드된 RPM 확인
ls ~/rpmbuild/RPMS/x86_64/
# myapp-1.0-1.x86_64.rpm
```

```
rpmbuild 옵션:
  -ba  = 바이너리 + 소스 RPM 모두 빌드
  -bb  = 바이너리 RPM 만 빌드
  -bs  = 소스 RPM 만 빌드
  -bp  = %prep 단계까지만 (압축 해제 테스트)
  -bc  = %build 단계까지 (컴파일 테스트)
```

---

---

# ⑧ rpm 명령어 한눈에

|명령어|역할|
|---|---|
|`rpm -q 패키지`|설치 여부 확인|
|`rpm -qa`|전체 설치 패키지 목록|
|`rpm -qi 패키지`|패키지 상세 정보|
|`rpm -qf 파일경로`|파일 → 패키지 역추적|
|`rpm -ql 패키지`|패키지가 설치한 파일 목록|
|`rpm -qR 패키지`|의존성 (필요한 것)|
|`rpm -q --whatrequires 패키지`|역의존성 (이 패키지 필요로 하는 것)|
|`rpm -qc 패키지`|설정 파일 목록|
|`rpm -V 패키지`|무결성 검증|
|`rpm -qVf 파일`|파일 소속 패키지 무결성 검증|
|`rpm -ivh 파일.rpm`|설치|
|`rpm -e 패키지`|삭제|
|`rpm -e --test 패키지`|삭제 시뮬레이션 ⭐️|
|`rpm2cpio 파일.rpm \| cpio -t`|RPM 내부 목록 확인|
|`rpm2cpio 파일.rpm \| cpio -idv`|RPM 파일 추출|

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|rpm 설치 시 의존성 에러|rpm 은 의존성 자동 처리 안 함|`dnf install` 사용|
|패키지 삭제했더니 다른 서비스 멈춤|역의존성 확인 안 함|`rpm -e --test` 먼저|
|rpm -V 결과가 많이 나옴|설정 파일 수정은 정상|`c` 플래그 붙은 건 설정 파일 → 정상 수정|
|파일 출처 모름|역추적 방법 모름|`rpm -qf 파일경로` 로 확인|
|dnf / yum 어느 걸 써야 하나|버전 혼동|RHEL 8+/Fedora → `dnf` / 구버전 → `yum`|