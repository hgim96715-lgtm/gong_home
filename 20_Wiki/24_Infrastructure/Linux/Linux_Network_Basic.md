---
aliases:
  - OSI 7계층
  - IP 주소
  - 서브넷
  - 게이트웨이
  - DNS
  - 네트워크 기초
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[CS_TCP_IP]]"
  - "[[CS_HTTP_Basics]]"
  - "[[Linux_Network_Commands]]"
  - "[[Linux_Port]]"
---

# Linux_Network_Basic — 네트워크 기초

## 한 줄 요약

```
데이터가 A 에서 B 로 어떻게 이동하는가
OSI 7계층 → IP → 서브넷 → 게이트웨이 → DNS → TCP/UDP
```

---

---

# ① OSI 7계층

>OSI :Open Systems Interconnection Reference Model

```
네트워크 통신을 7단계로 나눈 표준 모델
"모든 사람이 네트워킹을 편리하게 쓰도록"

7층  응용 계층   (Application)   HTTP / FTP(파일전송) / DNS / SSH / SMTP(전자우편) / TELNET(원격접속)
6층  표현 계층   (Presentation)  암호화(TLS) / 인코딩 / 압축 / 복호화 / 압축 해제
5층  세션 계층   (Session)       연결 수립 / 유지 / 종료
4층  전송 계층   (Transport)     TCP / UDP / 포트 번호 
3층  네트워크 계층(Network)        라우터 / IP
2층  데이터링크  (Data Link)      MAC 주소 / 스위치 / 브리지
1층  물리 계층   (Physical)       케이블 / 전기 신호(무선주파수 링크) ->허브 / 리피터
```

```
실무에서 자주 보는 계층:
  7층 HTTP       → 브라우저, API 호출
  4층 TCP/UDP    → 포트, 연결 방식
  3층 IP         → IP 주소, 라우팅
  2층 MAC        → 같은 네트워크 내 식별

TCP/IP 4계층 모델 (실용 버전):
  OSI 7계층을 4개로 단순화
  응용 (5~7층) / 전송 (4층) / 인터넷 (3층) / 네트워크 (1~2층)
```

---

---

# ② IP 주소

```
인터넷에서 컴퓨터를 식별하는 주소
32비트 (IPv4) → 4개 숫자 점으로 구분

192.168.0.1
  192.168.0  → 네트워크 주소 (같은 네트워크)
  .1         → 호스트 주소 (그 네트워크 안의 컴퓨터)
```

## 사설 IP vs 공인 IP

```
공인 IP (Public IP):
  인터넷에서 유일한 주소
  외부에서 접근 가능
  ex) 203.252.12.1

사설 IP (Private IP):
  내부 네트워크에서만 사용
  외부 인터넷에서 직접 접근 불가
  10.0.0.0/8
  172.16.0.0/12
  192.168.0.0/16   ← 가장 흔함 (공유기 내부)

Docker 컨테이너:
  172.18.0.x 대역 사설 IP 자동 할당
  같은 네트워크(hospital-network) 안에서 통신
```

---

---

# ③ 서브넷 (Subnet)

```
하나의 큰 네트워크를 작은 네트워크로 쪼갠 것
서브넷 마스크로 네트워크/호스트 범위 구분
```

## IP 주소를 도로명 주소로 이해하기

```
192.168.0.10

  192.168.0  = 서울특별시 강남구  (네트워크 주소)
  .10        = 테헤란로 10번지    (호스트 주소)

서브넷 = "어디까지가 구이고 어디부터가 번지수인가" 를 정하는 것
```

## /24 가 무슨 말인가

```
IP 주소 = 32비트 숫자

  192  .  168  .   0  .  10
11000000 10101000 00000000 00001010

/24 = 앞 24비트는 네트워크 / 나머지 8비트는 호스트

11000000 10101000 00000000 | 00001010
←──────── 네트워크 (24비트) ────────→ ←호스트(8비트)→
    192  .   168  .   0                .  10

8비트로 표현할 수 있는 숫자: 0 ~ 255 (총 256개)
그 중 0 (네트워크 주소) 과 255 (브로드캐스트) 제외
→ 실제 사용 가능: 1 ~ 254 (254대)
```

## 서브넷 마스크 — /24 를 십진수로 표현한 것

```
255 = 11111111 (8비트 전부 1)
0   = 00000000 (8비트 전부 0)

/24 = 255.255.255.0
    = 11111111.11111111.11111111.00000000
      ←──────── 1인 부분 = 네트워크 ────────→ ←0인 부분= 호스트→

1 인 자리 = "이 부분은 네트워크 주소 → 건드리지 마"
0 인 자리 = "이 부분이 호스트 번호 → 여기서 골라"
```

```
서브넷 마스크 정리:
  /24 = 255.255.255.0   뒤 8비트 = 호스트  → 최대 254대
  /16 = 255.255.0.0     뒤 16비트 = 호스트 → 최대 65,534대
  /8  = 255.0.0.0       뒤 24비트 = 호스트 → 최대 16,777,214대

  숫자가 작을수록 더 큰 네트워크
```

## 특수 주소 — 왜 2개가 빠지나

```
192.168.0.0/24 에서:

192.168.0.0    네트워크 자체를 가리키는 주소 (사용 불가)
               "강남구 전체" 를 가리킴 → 특정 집 주소로 못 씀

192.168.0.255  브로드캐스트 주소 (사용 불가)
               "강남구 모든 집에 동시에 편지 보내기"
               → 특정 컴퓨터 IP 로 못 씀

192.168.0.1 ~ 192.168.0.254  → 실제 컴퓨터에 할당 가능 (254대)
```

```
표기법 요약:
  192.168.0.0/24
  → 사용 가능 IP: 192.168.0.1 ~ 192.168.0.254

  192.168.0.0/16
  → 앞 16비트 = 네트워크
  → 뒤 16비트 = 호스트 (약 65,000대)
  → 사용 가능 IP: 192.168.0.1 ~ 192.168.255.254
```

---

---

# ④ 게이트웨이 (Gateway)

```
내부 네트워크에서 외부 네트워크로 나가는 출입구
"집에서 인터넷으로 나가는 공유기"

패킷이 목적지에 가는 과정:
  내 컴퓨터 (192.168.0.10)
    → 공유기/게이트웨이 (192.168.0.1)
      → ISP (인터넷 서비스 제공자)
        → 목적지 서버
```

```bash
# 게이트웨이 확인
ip route show
route -n

# 기본 게이트웨이
default via 192.168.0.1 dev eth0
# → 모든 외부 트래픽은 192.168.0.1 로 보냄
```

```
Docker 네트워크:
  docker에서 예를들면 hospital-network 의 게이트웨이 = 172.18.0.1
  컨테이너들이 외부 인터넷 접속 시 이 게이트웨이 경유
  API 호출 (apis.data.go.kr) 도 이 경로로 나감
```

---

---

# ⑤ DNS (Domain Name System)

```
도메인 이름 → IP 주소 로 변환하는 전화번호부

사람: "apis.data.go.kr 에 접속하고 싶어"
컴퓨터: IP 주소만 이해 → DNS 에 물어봄
DNS: "apis.data.go.kr = 211.252.xxx.xxx"
컴퓨터: 해당 IP 로 접속
```

```
DNS 조회 순서:
  1. 브라우저 캐시 확인
  2. OS 캐시 확인 (/etc/hosts)
  3. 공유기/로컬 DNS 확인
  4. ISP DNS 서버 확인
  5. 루트 DNS → 최상위 도메인 DNS → 권한 DNS
```

```bash
# DNS 조회
nslookup apis.data.go.kr
dig apis.data.go.kr

# /etc/hosts — 로컬 DNS (우선순위 높음)
cat /etc/hosts
# 127.0.0.1    localhost
# 172.18.0.3   kafka     ← Docker 컨테이너 내부 DNS
```

## nslookup 결과 해석

```bash
$ nslookup apis.data.go.kr

Server:   210.220.163.82    ← 내 맥북이 사용하는 DNS 서버 (공유기/ISP)
Address:  210.220.163.82#53 ← 53번 포트 (DNS 는 항상 UDP 53 포트)

Non-authoritative answer:   ← 이 DNS 서버가 직접 관리하는 게 아님
                               다른 곳에서 조회해서 캐시로 가져온 답변
Name:    apis.data.go.kr
Address: 223.130.169.165    ← 실제 IP 주소 (이 IP 로 HTTP 요청 감)
```

## dig 결과 해석

```bash
$ dig apis.data.go.kr

;; QUESTION SECTION:
;apis.data.go.kr.   IN  A
# IN = 인터넷 클래스 / A = IPv4 주소 조회 요청

;; ANSWER SECTION:
apis.data.go.kr.  320  IN  A  223.130.169.165
#                 ↑               ↑
#                 TTL (초)        실제 IP
#                 320초 동안 캐시에 보관

;; Query time: 17 msec   ← DNS 조회에 걸린 시간
;; SERVER: 210.220.163.82#53  ← 조회한 DNS 서버
```

```
TTL (Time To Live):
  이 DNS 응답을 320초 동안 캐시에 보관
  320초 후 만료 → 다시 DNS 서버에 물어봄
  TTL 이 낮으면 IP 변경이 빨리 반영됨
  TTL 이 높으면 캐시를 오래 씀 (DNS 서버 부하 줄임)

Non-authoritative answer:
  권한 있는 DNS (data.go.kr 담당 서버) 가 아닌
  중간 DNS 서버가 캐시해서 답한 것
  → 정확하지만 최신이 아닐 수 있음 (TTL 만큼)
```


```
Docker 내부 DNS:
  같은 네트워크의 서비스명이 DNS 로 등록됨
  kafka → 172.18.0.3 자동 해석
  postgres → 172.18.0.4 자동 해석
  → KAFKA_BOOTSTRAP_SERVERS=kafka:9092 가 동작하는 이유
```

---

---

# ⑥ TCP vs UDP

|항목|TCP|UDP|
|---|---|---|
|연결|3-Way Handshake|없음|
|신뢰성|✅ 재전송 보장|❌|
|순서 보장|✅|❌|
|속도|느림|빠름|
|사용|HTTP / Kafka / DB|DNS / 스트리밍|

```
TCP (Transmission Control Protocol):
  연결 수립 후 통신 → 안정적
  패킷 유실 시 재전송 → 신뢰성 보장
  HTTP / HTTPS / SSH / Kafka / PostgreSQL

UDP (User Datagram Protocol):
  연결 없이 바로 전송 → 빠름
  유실되어도 재전송 안 함
  DNS 조회 / 스트리밍 / 온라인 게임
```

> 3-Way Handshake 상세 → [[CS_TCP_IP]] 참고

---

---

# ⑦ DE 관점 — 왜 알아야 하는가

```
Kafka:
  TCP 9092 포트
  Producer → Broker → Consumer 모두 TCP 연결
  KAFKA_ADVERTISED_LISTENERS 설정 → Docker DNS 활용

PostgreSQL:
  TCP 5432 포트
  DataGrip / Spark / Airflow 모두 TCP 로 연결

Docker Compose 네트워크:
  같은 network 블록 → 같은 서브넷
  서비스명 = DNS 이름 (kafka / postgres / spark-master)
  포트 포워딩 = 호스트 IP:포트 → 컨테이너 IP:포트

API 호출 (공공데이터):
  내 코드 → DNS 조회 → IP 확인
  → TCP 연결 (3-Way Handshake)
  → HTTP GET 요청 전송
  → XML 응답 수신
  → TCP 연결 종료
```