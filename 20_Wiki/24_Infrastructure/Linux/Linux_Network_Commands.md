---
aliases:
  - 네트워크 명령어
  - ifconfig
  - ip addr
  - netstat
  - ss
  - nmap
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Network_Basic]]"
  - "[[Linux_SSH]]"
  - "[[Linux_Port]]"
  - "[[Linux_Firewall]]"
---

# Linux_Network_Commands — 네트워크 명령어

## 한 줄 요약

```
ifconfig / ip addr  → 내 네트워크 인터페이스 확인
ping / traceroute   → 연결 상태 / 경로 확인
netstat / ss        → 포트 / 소켓 상태 확인
nmap                → 원격 포트 스캔
```

---

---

# ① ifconfig — 네트워크 인터페이스 (구버전)

```
구버전 도구 (net-tools 패키지)
Ubuntu 18+ / RHEL 8+ 에서 기본 제거
→ ip 명령어로 대체 권장
하지만 여전히 많은 서버에서 사용 중
```

```bash
# 설치 (없으면)
sudo apt-get install net-tools

# 전체 인터페이스 목록
ifconfig

# 특정 인터페이스만
ifconfig eth0

# 인터페이스 활성화 / 비활성화
sudo ifconfig eth0 up
sudo ifconfig eth0 down

# IP 임시 변경
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0
```

## ifconfig 출력 해석

```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 172.16.50.3  netmask 255.255.240.0  broadcast 172.16.63.255
      ↑ IPv4 주소        ↑ 서브넷 마스크          ↑ 브로드캐스트
      inet6 fe80::...   prefixlen 64
      ↑ IPv6 주소
      ether 02:42:ac:10:32:03   txqueuelen 1000
      ↑ MAC 주소 (하드웨어 주소)

lo: flags=73<UP,LOOPBACK,RUNNING>
    inet 127.0.0.1   ← 루프백 (자기 자신)
```

---

---

# ② ip — 네트워크 인터페이스 (현대 도구) ⭐️

```
ifconfig 의 현대적 대체
iproute2 패키지 (기본 설치)
더 많은 기능 / 더 일관된 문법
```

## ip addr — IP 주소 확인

```bash
ip addr              # 전체 인터페이스
ip addr show eth0    # 특정 인터페이스
ip a                 # 축약형

# IP만 깔끔하게 추출
hostname -I          # 모든 IP 한 줄로
ip addr | grep "inet " | awk '{print $2}'
```

## ip link — 인터페이스 상태

```bash
ip link show         # 전체 인터페이스 링크 상태
ip link show eth0

# 인터페이스 활성화 / 비활성화
sudo ip link set eth0 up
sudo ip link set eth0 down
```

## ip route — 라우팅 테이블

```bash
ip route show        # 라우팅 테이블 전체
ip route             # 축약형

# 특정 IP 로 가는 경로
ip route get 8.8.8.8

# 기본 게이트웨이 확인
ip route | grep default
# default via 172.16.48.1 dev eth0
```

## ifconfig vs ip 비교

|기능|ifconfig (구)|ip (신)|
|---|---|---|
|인터페이스 목록|`ifconfig`|`ip addr`|
|특정 인터페이스|`ifconfig eth0`|`ip addr show eth0`|
|링크 상태|`ifconfig eth0 up`|`ip link set eth0 up`|
|라우팅|`route`|`ip route`|

---

---

# ③ ping — 연결 확인 ⭐️

```
ICMP Echo Request/Reply 를 보내서
목적지 서버가 살아있는지, 응답 시간이 어떤지 확인
```

```bash
# 기본 사용
ping 8.8.8.8           # 구글 DNS (무한 반복)
ping google.com        # 도메인으로도 가능

# 횟수 제한
ping -c 4 8.8.8.8      # 4번만 보내고 종료

# 간격 지정
ping -i 2 8.8.8.8      # 2초 간격

# 패킷 크기 지정
ping -s 1400 8.8.8.8   # 1400바이트 패킷
```

## ping 출력 해석

```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=1.23 ms
                                             ↑ 응답 시간 (낮을수록 좋음)

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
                               ↑ 패킷 손실률 (0%=정상)
rtt min/avg/max/mdev = 1.1/1.2/1.4/0.1 ms
    ↑ 최소/평균/최대/편차
```

```
ping 실패 유형:
  Request timeout      → 응답 없음 (방화벽 or 서버 다운)
  Destination unreachable → 경로 없음 (라우팅 문제)
  100% packet loss     → 완전 차단 또는 서버 다운
```

---

---

# ④ traceroute — 경로 추적 ⭐️

```
패킷이 목적지까지 가는 동안
거치는 라우터(홉) 를 순서대로 보여줌
어느 구간에서 지연이 생기는지 파악
```

```bash
# 설치 (없으면)
sudo apt-get install traceroute

# 기본 사용
traceroute google.com
traceroute 8.8.8.8

# UDP 대신 ICMP 사용 (방화벽 우회 시)
traceroute -I 8.8.8.8

# 홉 수 제한
traceroute -m 15 google.com   # 최대 15홉
```

## traceroute 출력 해석

```
traceroute to google.com (142.250.196.110)
 1  _gateway (172.16.48.1)   0.5 ms   0.4 ms   0.5 ms
 ↑ 첫 번째 홉 (게이트웨이)
 2  203.248.xx.xx            2.1 ms   1.9 ms   2.0 ms
 3  * * *
 ↑ 응답 없음 (방화벽에서 ICMP 차단)
 4  142.250.xx.xx           15.2 ms  14.8 ms  15.1 ms
 ...
12  google.com              16.8 ms  16.5 ms  16.9 ms

* * * = 해당 홉이 응답 안 함 (정상일 수 있음)
```

---

---

# ⑤ netstat — 소켓 / 포트 상태 (구버전)

```
구버전 도구 (net-tools)
ss 명령어로 대체 권장
여전히 많이 쓰임
```

```bash
# 설치
sudo apt-get install net-tools

# 모든 연결 + 포트
netstat -an

# 리스닝 포트만 (프로그램 이름 포함)
sudo netstat -tulnp
#             ↑↑↑↑↑
#             t = TCP
#             u = UDP
#             l = Listening (대기 중)
#             n = 숫자로 (도메인 변환 없이)
#             p = 프로그램명/PID

# TCP 연결만
netstat -tn

# 통계
netstat -s

# 라우팅 테이블
netstat -r
```

## netstat -tulnp 출력 해석

```
Proto  Local Address      Foreign Address   State   PID/Program
tcp    0.0.0.0:22         0.0.0.0:*         LISTEN  1234/sshd
tcp    0.0.0.0:5432       0.0.0.0:*         LISTEN  5678/postgres
tcp    127.0.0.1:6379     0.0.0.0:*         LISTEN  9012/redis
udp    0.0.0.0:68         0.0.0.0:*                 1111/dhclient

0.0.0.0:22    → 모든 인터페이스의 22번 포트 (외부 접근 가능)
127.0.0.1:6379 → 루프백만 (로컬에서만 접근 가능)
```

---

---

# ⑥ ss — 소켓 통계 (현대 도구) ⭐️

```
netstat 의 현대적 대체
iproute2 패키지 (기본 설치)
더 빠름 / 더 많은 정보
```

```bash
# 리스닝 포트 (netstat -tulnp 와 동일)
ss -tulnp

# TCP 연결만
ss -tn

# 특정 포트 필터
ss -tulnp | grep :22      # 22번 포트
ss -tulnp | grep LISTEN   # 리스닝 상태만

# 상태별 확인
ss -tn state established   # 연결 확립된 것만
ss -tn state time-wait     # TIME_WAIT 상태

# 프로세스별 소켓
ss -tp
```

## netstat vs ss 비교

|기능|netstat (구)|ss (신)|
|---|---|---|
|리스닝 포트|`netstat -tulnp`|`ss -tulnp`|
|속도|느림|빠름|
|상세 필터|제한적|`state`, `sport`, `dport` 등|
|설치|net-tools|기본 설치|

---

---

# ⑦ nmap — 포트 스캔 ⭐️

```
원격 서버의 열린 포트 / 서비스 스캔
네트워크 보안 점검 / 방화벽 검증
⚠️ 허가된 서버에만 사용 (무단 스캔 = 불법)
```

```bash
# 설치
sudo apt-get install nmap

# 기본 포트 스캔 (상위 1000개 포트)
nmap 192.168.1.10
nmap google.com

# 전체 포트 스캔 (1~65535)
nmap -p- 192.168.1.10

# 특정 포트만
nmap -p 22,80,443 192.168.1.10
nmap -p 1-1000 192.168.1.10

# 서비스/버전 감지
nmap -sV 192.168.1.10

# OS 감지
sudo nmap -O 192.168.1.10

# 빠른 스캔
nmap -F 192.168.1.10

# 대역 전체 스캔
nmap 192.168.1.0/24   # 서브넷 전체
```

## nmap 출력 해석

```
PORT     STATE    SERVICE   VERSION
22/tcp   open     ssh       OpenSSH 8.9
80/tcp   open     http      nginx 1.22
443/tcp  open     https
3306/tcp filtered mysql
8080/tcp closed   http-proxy

open     = 포트 열림 (서비스 실행 중)
closed   = 포트 닫힘 (서비스 없음)
filtered = 방화벽에서 차단
```

---

---

# ⑧ 실전 트러블슈팅 패턴

## Docker 컨테이너 포트 확인

```bash
# 호스트에서 열린 포트
ss -tulnp | grep docker
ss -tulnp | grep :8080

# 특정 포트 리스닝 여부
ss -tulnp | grep ":5432"   # PostgreSQL
ss -tulnp | grep ":9092"   # Kafka
ss -tulnp | grep ":8888"   # Jupyter
```

## 서비스 포트 확인 흐름

```
1. 서비스 접속 안 됨
      ↓
2. ping 서버IP          → 서버 살아있나?
      ↓ 응답 있음
3. ss -tulnp | grep 포트 → 포트 열려있나?
      ↓ 포트 열림
4. sudo ufw status      → 방화벽 차단?
      ↓ 방화벽 OK
5. nmap -p 포트 서버IP  → 외부에서 접근 가능?
```

## 네트워크 연결 끊겼을 때

```bash
# 1. 인터페이스 상태
ip link show
ip addr

# 2. 게이트웨이 핑
ping $(ip route | grep default | awk '{print $3}')

# 3. DNS 확인
cat /etc/resolv.conf
ping 8.8.8.8        # IP 로 핑 (DNS 우회)
ping google.com     # 도메인으로 핑 (DNS 확인)
```

---

---

# 명령어 한눈에

|목적|명령어|비고|
|---|---|---|
|내 IP 확인|`ip addr` / `hostname -I`|`ifconfig` 구버전|
|게이트웨이 확인|`ip route \| grep default`||
|연결 테스트|`ping -c 4 대상`||
|경로 추적|`traceroute 대상`||
|열린 포트 확인|`ss -tulnp`|`netstat -tulnp` 구버전|
|원격 포트 스캔|`nmap -p- 서버IP`||
|특정 포트 확인|`ss -tulnp \| grep :포트`||

---

---

# 자주 하는 실수

|실수|원인|해결|
|---|---|---|
|`ifconfig` 없음|net-tools 미설치|`sudo apt install net-tools`|
|ping 응답 없음|ICMP 차단일 수 있음|`nmap -sn` 으로 호스트 탐지|
|netstat 느림|구버전 도구|`ss` 로 대체|
|nmap 느림|전체 포트 스캔|`-F` 옵션으로 빠른 스캔|
|traceroute `* * *` 연속|ICMP 차단|`traceroute -I` (ICMP) 사용|