---
aliases:
  - 소스 컴파일
  - configure
  - make
  - make install
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Package_APT]]"
  - "[[Linux_Package_RPM_YUM_DNF]]"
  - "[[Linux_Archive]]"
---

# Linux_Compile_Install — 소스 코드 컴파일 설치

## 한 줄 요약

```
패키지 매니저(apt/yum) 로 설치 불가한 소프트웨어를
소스 코드를 직접 컴파일해서 설치하는 방법

흐름:
  소스 다운로드 → 압축 해제 → configure → make → make install
```

---

---

# ① 왜 소스 컴파일이 필요한가

```
패키지 매니저로 해결 안 되는 상황:
  저장소에 없는 최신 버전 필요
  특정 옵션으로 커스텀 빌드 필요
  폐쇄망 환경 (인터넷 없음)
  패키지가 없는 희귀 소프트웨어

단점:
  과정이 복잡 (에러 많음)
  패키지 매니저처럼 자동 업데이트 없음
  제거 시 수동으로 파일 추적 필요
  → 가능하면 apt/yum 우선 사용
```

## 패키지 매니저 vs 소스 컴파일 비교

```
패키지 매니저 (apt/yum):
  장점: 간단 / 자동 의존성 / 자동 업데이트
  단점: 저장소 버전에 제한 / 커스텀 옵션 불가
  설치 위치: /usr/bin

소스 컴파일:
  장점: 최신 버전 / 커스텀 옵션 / 어디서나 가능
  단점: 복잡 / 시간 오래 걸림 / 수동 관리
  설치 위치: /usr/local/bin (기본)
```

---

---

# ② 전체 흐름 ⭐️

```
1. 소스 다운로드       wget / curl / git clone
2. 압축 해제          tar -zxvf
3. 소스 디렉토리 진입  cd 소스폴더
4. 환경 확인          ./configure
5. 컴파일             make
6. 설치              sudo make install
7. 확인              which 프로그램명
8. (필요 시) 제거     sudo make uninstall
```

---

---

# ③ 소스 다운로드 & 압축 해제

## 다운로드

```bash
# wget 으로 다운로드
wget https://example.com/pure-ftpd-1.0.53.tar.gz

# curl 로 다운로드
curl -O https://example.com/pure-ftpd-1.0.53.tar.gz

# git 으로 소스 클론
git clone https://github.com/user/project.git
```

## 압축 해제

```bash
ls -l                              # 다운로드된 파일 확인
tar -zxvf pure-ftpd-1.0.53.tar.gz
#      ↑↑↑↑
#      z = gzip 압축 해제
#      x = extract (추출)
#      v = verbose (파일 목록 출력)
#      f = 파일명 지정

ls -l   # 새 디렉토리 생성됨 확인
cd pure-ftpd-1.0.53   # 소스 폴더로 이동
ls      # Makefile.in / configure / src/ 등 확인
```

```
-v 옵션을 붙이는 이유:
  압축 해제 중 어떤 파일이 풀리는지 확인
  중간에 에러 / 용량 부족 / 권한 문제 발생 시
  어느 파일에서 멈췄는지 즉시 파악 가능
```

---

---

# ④ ./configure — 환경 확인 & Makefile 생성 ⭐️

```
./configure 가 하는 일:
  시스템에 필요한 도구/라이브러리가 있는지 검사
  운영 환경에 맞는 Makefile 자동 생성

config.status: creating Makefile
  ↑ 이 줄 나오면 성공 ✅

실패하면:
  에러 메시지 확인 → 부족한 패키지 apt/yum 으로 설치 → 재시도
```

## 기본 실행

```bash
./configure
# 시스템 환경 검사 중...
# checking for gcc... gcc
# checking for make... make
# ...
# config.status: creating Makefile  ← 성공
```

## 주요 옵션

```bash
# 설치 경로 지정 (기본: /usr/local)
./configure --prefix=/opt/myapp

# 특정 기능 켜기/끄기
./configure --enable-ssl          # SSL 활성화
./configure --disable-ipv6        # IPv6 비활성화
./configure --with-openssl        # OpenSSL 연동

# 도움말 (사용 가능한 옵션 전체 확인)
./configure --help
```

```
--prefix 가 중요한 이유:
  기본: /usr/local/bin, /usr/local/lib, /usr/local/etc
  커스텀: --prefix=/opt/myapp
          → /opt/myapp/bin, /opt/myapp/lib

  나중에 make uninstall 없이 지우려면:
  rm -rf /opt/myapp  한 줄로 끝
  → 관리하기 쉬움
```

## configure 에러 대처 ⭐️

```bash
# 에러 예시
# checking for openssl... no
# configure: error: OpenSSL not found

# 해결: 필요한 개발 라이브러리 설치
sudo apt-get install libssl-dev     # Debian 계열
sudo yum install openssl-devel      # Red Hat 계열

# 재실행
./configure
```

```
자주 나오는 에러:
  gcc not found     → sudo apt install build-essential
  make not found    → sudo apt install make
  라이브러리 없음   → sudo apt install 라이브러리-dev
  권한 없음         → sudo 사용

개발 라이브러리 이름:
  Debian: openssl → libssl-dev  (lib + 이름 + -dev)
  RHEL:   openssl → openssl-devel (이름 + -devel)
```

---

---

# ⑤ make — 컴파일 ⭐️

```
make 가 하는 일:
  Makefile 을 읽고
  gcc(C 컴파일러) 등을 올바른 옵션으로 실행
  소스 코드(.c) → 실행 파일(바이너리) 변환

시간이 오래 걸릴 수 있음 (프로젝트 규모에 따라 수분~수십분)
에러 없이 완료되면 컴파일 성공
```

```bash
# 기본 컴파일
make

# 병렬 컴파일 (CPU 코어 수만큼 빠르게)
make -j4        # 4개 코어 병렬 사용
make -j$(nproc) # 자동으로 코어 수 감지 (권장)

# 특정 타겟만 빌드
make clean      # 이전 빌드 결과물 삭제 (재빌드 시)
make all        # 전체 빌드 (기본값과 동일)
```

```
make -j$(nproc) 권장 이유:
  nproc = 현재 시스템 CPU 코어 수 출력
  자동으로 최적 병렬 수 설정
  빌드 시간 크게 단축
```

## make 에러 대처

```bash
# 에러 발생 시
# 에러 메시지 확인 → 원인 파악 → 해결 후 재시도

# 이전 빌드 결과물 삭제 후 처음부터
make clean
./configure
make
```

---

---

# ⑥ sudo make install — 설치

```
컴파일된 바이너리를 시스템 경로에 복사
/usr/local/bin, /usr/local/sbin 등에 설치
시스템 디렉토리에 쓰기 → sudo 필요
```

```bash
sudo make install

# 설치 확인
which pure-ftpd
# /usr/local/bin/pure-ftpd  ← 소스 컴파일 설치 경로

which python3
# /usr/bin/python3           ← 패키지 매니저 설치 경로

# 버전 확인
pure-ftpd --version
```

```
설치 경로 차이:
  패키지 매니저 → /usr/bin/
  소스 컴파일   → /usr/local/bin/

  /usr/local/ 이 "내가 직접 설치한 것" 전용 공간
  시스템 패키지와 구분됨
```

---

---

# ⑦ sudo make uninstall — 제거

```
make install 의 반대 작업
설치된 파일들을 시스템에서 제거

⚠️ 원본 소스 폴더와 Makefile 이 있어야 작동
   설치 후 소스 폴더 지우면 uninstall 불가
   → 소스 폴더는 uninstall 전까지 보존
```

```bash
# 소스 폴더 안에서 실행
cd ~/project/pure-ftpd-1.0.53
sudo make uninstall

# 확인
which pure-ftpd
# (아무 출력 없음) ← 제거됨

# 소스 폴더도 삭제
cd ~/project          # 소스 폴더 밖으로 먼저!
rm -r pure-ftpd-1.0.53
ls -l
```

```
소스 폴더 보존 원칙:
  make install 직후 소스 삭제 → make uninstall 불가
  나중에 수동으로 파일 찾아서 하나씩 삭제해야 함

  해결책:
  1. 소스 폴더 보존 (uninstall 까지)
  2. --prefix=/opt/앱이름 설정 → rm -rf /opt/앱이름 으로 삭제
  3. checkinstall 도구 사용 (deb/rpm 패키지로 변환 후 설치)
```

---

---

# ⑧ checkinstall — 소스를 패키지로 변환 ⭐️

```
소스 컴파일의 단점:
  make uninstall 에 의존 (소스 폴더 필요)
  패키지 관리자로 추적 불가

checkinstall 을 쓰면:
  make install 대신 checkinstall 실행
  → .deb 또는 .rpm 패키지 자동 생성 + 설치
  → apt/yum 으로 추적 및 제거 가능
```

```bash
# 설치
sudo apt-get install checkinstall   # Debian 계열
sudo yum install checkinstall       # Red Hat 계열

# make install 대신 checkinstall 사용
sudo checkinstall

# 이후 패키지 매니저로 제거 가능
sudo apt remove 패키지명
sudo rpm -e 패키지명
```

---

---

# ⑨ 전체 실전 예시

```bash
# pure-ftpd 1.0.53 소스 컴파일 설치

# 1. 다운로드 및 압축 해제
wget https://example.com/pure-ftpd-1.0.53.tar.gz
tar -zxvf pure-ftpd-1.0.53.tar.gz
cd pure-ftpd-1.0.53

# 2. 환경 확인 (에러 나면 필요 패키지 설치 후 재시도)
./configure --prefix=/usr/local

# 3. 컴파일 (병렬로 빠르게)
make -j$(nproc)

# 4. 설치
sudo make install

# 5. 확인
which pure-ftpd
pure-ftpd --version

# 6. 제거 (필요 시)
sudo make uninstall
cd ..
rm -r pure-ftpd-1.0.53
```

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`./configure` 에러|필요 라이브러리 없음|에러 메시지 확인 → apt/yum 으로 설치|
|`./` 없이 `configure` 실행|PATH 에 없음|반드시 `./configure`|
|`make install` 에 sudo 없음|시스템 폴더 쓰기 권한 없음|`sudo make install`|
|설치 후 소스 폴더 삭제|용량 확보|`make uninstall` 전까지 보존|
|`make uninstall` 안 됨|소스 폴더 없음|`--prefix` 로 설치 → `rm -rf` 로 제거|
|컴파일 느림|단일 코어|`make -j$(nproc)` 병렬 빌드|
|`configure: error: gcc not found`|빌드 도구 없음|`sudo apt install build-essential`|