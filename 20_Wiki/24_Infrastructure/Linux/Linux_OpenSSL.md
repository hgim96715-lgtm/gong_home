---
aliases:
  - openssl
  - 키 생성
  - SECRET_KEY
  - 인증서
  - base64
  - SHA256
tags:
  - Linux
related:
  - "[[SSH_Connection]]"
  - "[[File_Permissions]]"
  - "[[00_Linux_HomePage]]"
---

# Linux_OpenSSL — 키 생성 · 인증서 · 암호화

## 한 줄 요약

```
SSL/TLS 인증서 생성, 비밀키 생성, 해시 계산 등
보안 관련 작업을 터미널에서 처리하는 도구
```

---

---

# ① rand — 랜덤 키 생성 ⭐️

```
openssl rand -base64 N   → N바이트 랜덤값을 base64 문자열로 출력
openssl rand -hex N      → N바이트 랜덤값을 16진수로 출력

주 용도:
  Superset SECRET_KEY 생성
  Airflow WEBSERVER__SECRET_KEY 생성
  JWT 시크릿 키 생성
  .env 에 들어가는 비밀값 생성
```

```bash
# base64 형식 (영문+숫자+특수문자, 가장 많이 씀)
openssl rand -base64 42
# 결과 예: kJ3mP9xQ2vR8tN5wL1aB7yC0dE6fG4hI+jK=

# hex 형식 (16진수, 숫자+a-f 만)
openssl rand -hex 32
# 결과 예: 4a2f8c1e9b3d7f0e5a2c6b8d1e4f7a0c

# 32바이트 → 더 짧은 키
openssl rand -base64 32

# 생성 즉시 .env 에 저장
echo "SECRET_KEY=$(openssl rand -base64 42)" >> .env
```

```
base64 vs hex:
  base64  더 짧은 문자열, 특수문자 포함 (+, /, =)
  hex     더 긴 문자열, 숫자+소문자만 → URL/설정파일 안전
```

## 실전 — Superset / Airflow SECRET_KEY 생성

```bash
# 1. 키 생성
openssl rand -base64 42

# 2. .env 에 복붙
SUPERSET_SECRET_KEY=kJ3mP9xQ2vR8tN5wL1aB7yC0dE6fG4hI+jK=
AIRFLOW_SECRET_KEY=anotherRandomKey...

# 3. docker-compose.yml 에서 ${변수명} 으로 주입
environment:
  SUPERSET_SECRET_KEY: ${SUPERSET_SECRET_KEY}
```

---

---

# ② dgst — 해시 생성 (SHA256 등)

```
파일이나 문자열의 해시값 계산
파일 무결성 확인, 비밀번호 검증 등에 사용
```

```bash
# 문자열 SHA256 해시
echo -n "hello" | openssl dgst -sha256
# SHA256(stdin)= 2cf24dba5fb0a30...

# 파일 SHA256 해시
openssl dgst -sha256 myfile.txt

# MD5 (보안용도보단 체크섬 용도)
openssl dgst -md5 myfile.txt

# 결과값만 출력 (앞 라벨 제거)
echo -n "hello" | openssl dgst -sha256 | awk '{print $2}'
```

---

---

# ③ enc — 파일 암호화 / 복호화

```
openssl enc 옵션 분해:
  -aes-256-cbc   암호화 알고리즘 (AES 256비트, CBC 모드)
  -in            입력 파일 (원본)
  -out           출력 파일 (결과물 저장 경로)
  -k             비밀번호 직접 입력
  -d             복호화 모드 (decrypt, 없으면 기본 암호화)
  -pbkdf2        비밀번호 → 키 변환 알고리즘 (최신 방식, 권장)
  -iter 100000   키 변환 반복 횟수 (높을수록 안전)
```


```bash
# 파일 암호화
openssl enc -aes-256-cbc -pbkdf2 -iter 100000 \
  -in  plaintext.txt \   # 원본 파일
  -out encrypted.bin \   # 암호화된 결과물 저장
  -k   mypassword        # 비밀번호

# 복호화 (-d 옵션 추가)
openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 \
  -in  encrypted.bin \   # 암호화된 파일
  -out decrypted.txt \   # 복호화된 결과물 저장
  -k   mypassword        # 암호화할 때 쓴 비밀번호와 동일해야 함
```

```
암호화 알고리즘 선택:
  -aes-256-cbc   현재 가장 많이 쓰는 표준 방식
  -aes-128-cbc   더 빠름, 보안은 약간 낮음
  256 > 128 (숫자 클수록 키 길이 길어서 더 안전)

출력 파일 확장자:
  .bin   바이너리 (이진) 형태 — 사람이 읽을 수 없음
  .enc   관행적으로 암호화 파일에 붙이는 확장자
  사실 어떤 확장자든 상관없음 — 내용이 암호화된 것
```

---

---

# ④ 인증서 관련

## 인증서(Certificate) 가 뭔가?

```
인증서 = "나는 진짜 example.com 입니다" 를 증명하는 신분증
  → 브라우저가 https:// 로 접속할 때 이걸 확인
  → 만료일이 지나면 "보안 경고" 뜸

구성 요소:
  key.pem   개인키 (비밀 — 절대 공개 금지)
  cert.pem  인증서 (공개 — 서버가 브라우저에 보내는 것)
```

## 도메인 SSL 인증서 확인


```bash
# 도메인의 인증서 정보 확인 (만료일 등)
openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -dates
```

```
옵션 분해:
  s_client              SSL 클라이언트로 서버에 접속
  -connect 도메인:포트  접속할 서버 (443 = HTTPS 기본 포트)
  2>/dev/null           접속 로그(에러) 는 버림 — 인증서 정보만 남김
  | openssl x509        파이프로 인증서 파싱
  -noout                인증서 원문은 출력 안 함
  -dates                만료일만 출력

출력 예시:
  notBefore=Jan  1 00:00:00 2025 GMT   ← 인증서 시작일
  notAfter=Jan   1 00:00:00 2026 GMT   ← 만료일 ← 이게 핵심
```

## 자체 서명 인증서 생성 — 로컬 개발용

```
공식 인증서는 Let's Encrypt 같은 기관에서 발급
로컬 개발환경에서는 직접 만든 자체 서명(Self-Signed) 인증서 사용

단: 브라우저에서 "신뢰할 수 없는 인증서" 경고 뜸
    → 로컬 개발 / 테스트 용도로만 사용
```

```bash
openssl req -x509 -newkey rsa:4096 \
  -keyout key.pem \    # 개인키 저장 경로
  -out    cert.pem \   # 인증서 저장 경로
  -days   365 \        # 유효 기간 (365일)
  -nodes               # 개인키에 비밀번호 없이 저장 (no DES)
```

```
옵션 분해:
  req           인증서 요청(CSR) 생성
  -x509         CSR 대신 자체 서명 인증서 바로 생성
  -newkey rsa:4096  RSA 4096비트 새 키 생성
                    (2048도 가능, 4096이 더 안전)
  -keyout       개인키 저장할 파일명
  -out          인증서 저장할 파일명
  -days         인증서 유효 기간 (일수)
  -nodes        no DES = 개인키 비밀번호 없음
                (없으면 key.pem 열 때마다 비밀번호 요구)

실행하면 국가/조직명 등 입력 프롬프트 나옴:
  Country Name: KR
  Organization: MyCompany
  Common Name: localhost   ← 접속할 도메인/IP
  (전부 그냥 Enter 눌러도 됨)
```

## 인증서 내용 확인


```bash
# 인증서 상세 정보 출력
openssl x509 -in cert.pem -noout -text
```

```
옵션 분해:
  x509         인증서 파싱 도구
  -in          읽을 인증서 파일
  -noout       인증서 원문(base64) 출력 안 함
  -text        사람이 읽기 좋은 형태로 출력

주요 확인 항목:
  Subject      인증서 소유자 (CN= 부분이 도메인)
  Issuer       발급자 (자체 서명이면 Subject = Issuer)
  Not Before   시작일
  Not After    만료일
  Public Key   공개키 정보
```

## .pem 파일이 뭔가?

```
PEM (Privacy Enhanced Mail)
  = base64 로 인코딩된 인증서/키 파일 형식
  텍스트 파일이라서 cat 으로 열면 내용 보임

-----BEGIN CERTIFICATE-----        ← 인증서
MIIFazCCA1OgAwIBAgI...
-----END CERTIFICATE-----

-----BEGIN PRIVATE KEY-----        ← 개인키
MIIEvQIBADANBgkqhkiG9w0BAQEF...
-----END PRIVATE KEY-----

key.pem   개인키  — 절대 공개 금지, .gitignore 추가 필수
cert.pem  인증서  — 공개해도 됨 (서버가 클라이언트에 전송)
```

---

---

# 자주 쓰는 명령어 치트시트

|명령어|용도|
|---|---|
|`openssl rand -base64 42`|SECRET_KEY 생성 (Superset, Airflow)|
|`openssl rand -hex 32`|16진수 랜덤 토큰 생성|
|`openssl dgst -sha256 파일`|파일 무결성 확인|
|`echo -n "값" \| openssl dgst -sha256`|문자열 해시|
|`openssl s_client -connect 도메인:443`|SSL 인증서 확인|

---

---

# ⚠️ 주의사항

```
openssl rand 결과는 실행할 때마다 다름
  → 한 번 생성하면 .env 에 저장해서 재사용

base64 결과에 = 가 끝에 붙을 수 있음 (패딩)
  → 일부 설정에서 = 제거가 필요하면:
  openssl rand -base64 42 | tr -d '='

SECRET_KEY 는 절대 git 에 올리면 안 됨
  → .env 를 .gitignore 에 반드시 추가
```