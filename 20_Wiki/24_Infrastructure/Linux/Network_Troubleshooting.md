---
aliases:
  - ping
  - traceroute
  - nslookup
  - dig
  - 네트워크 진단
  - DNS 확인
tags:
  - Linux
related:
  - "[[Network_Status]]"
  - "[[File_Transfer]]"
---
## 개념 한 줄 요약

**"네트워크의 의사 선생님들: 청진기(`ping`), 내시경(`traceroute`), 그리고 주소록 확인(`nslookup`/`dig`)."**

* **`ping`:** 대상이 살아있는지, 응답 속도(Latency)는 어떤지 확인한다. (ICMP 프로토콜 사용) 
* **`traceroute`:** 목적지까지 가는 경로 중 어디서 막히는지 추적한다. (병목 구간 확인) 
* **`nslookup` / `dig`:** 도메인 주소(google.com)가 어떤 IP로 연결되는지(DNS) 확인한다. 

---
##  왜 필요한가? (Why)

**문제점:**
- "웹사이트에 접속이 안 돼요. 서버가 죽었나요?"
- "특정 지역에서만 사이트가 엄청 느려요."
- "도메인을 샀는데 아직 연결이 안 된 것 같아요."

**해결책:**
- **`ping`** 으로 서버가 아예 죽었는지(응답 없음), 느린지(시간 지연) 1차 진단. 
- **`traceroute`** 로 "우리 집 공유기 문제인지, 통신사 문제인지, 서버 앞단 방화벽 문제인지" 위치 파악.
- **`dig`** 로 DNS 설정이 전 세계에 잘 전파됐는지 확인. 

---
## 실무 적용 시나리오 (Diagnosis Workflow)

**상황: "Google.com이 안 열려요!"**

1.  **연결 확인 (`ping`):** "일단 구글 서버가 살아있나?"
    * `ping google.com` -> 응답 없으면 인터넷 문제거나 서버 문제.
2.  **경로 추적 (`traceroute`):** "어디서 끊겼지?"
    * `traceroute google.com` -> 1번(공유기)부터 막히면 내 문제, 중간에서 막히면 통신사 문제.
3.  **DNS 확인 (`dig`):** "혹시 엉뚱한 주소로 가고 있나?"
    * `dig google.com` -> IP가 이상하게 나오면 DNS 설정 문제.

---
##  Code Core Points: ① `ping` (생존 신고) 

가장 먼저 해보는 기초 검사다. CMP 패킷을 보내서 돌아오는 시간을 잰다. 

```bash
# 1. 기본 사용 (계속 보냄 - 멈추려면 Ctrl+C)
ping google.com

# 2. 횟수 지정 (-c) ⭐️ [Mac/Linux 필수]
# "딱 5번만 보내고 멈춰." (스크립트 짤 때 유용)
ping -c 5 google.com
```

**결과 해석**

```text
실행후 나오는 것 
64 bytes from ... time=48.9 ms
64 bytes from ... time=56.5 ms
...
^C
종료후 나오는 것
--- google.com ping statistics ---
7 packets transmitted, 7 received, 0% packet loss, time 6006ms
rtt min/avg/max/mdev = 48.9/55.8/72.9/7.4 ms
```

|항목|의미|해석|
|---|---|---|
|`time=48.9 ms`|왕복 시간 (RTT)|낮을수록 빠름 (국내 <10ms, 해외 ~50~150ms)|
|`icmp_seq`|패킷 번호|번호가 끊기면 패킷 손실 의심|
|`ttl`|홉 수(Time To Live)|너무 작으면 경로 문제 / 루프 가능성|
|`packets transmitted`|보낸 패킷 수|내가 보낸 ping 개수|
|`received`|받은 응답 수|서버가 응답한 개수|
|`packet loss`|패킷 손실률|**0%가 정상**, 있으면 네트워크 불안정|
|`rtt min/avg/max`|응답 시간 통계|최소 / 평균 / 최대 RTT|

---
## Code Core Points: ② `traceroute` (경로 추적) 

출발지부터 목적지까지 거치는 모든 라우터(경유지)를 보여준다. 
TTL(Time To Live) 값을 1씩 늘려가면서, 각 관문마다 "너 누구니?" 하고 묻는 원리다.

> Ubuntu(특히 최소 이미지, 컨테이너)는 `traceroute`가 **기본 설치 안 돼 있음**.
> sudo apt update
> sudo apt install traceroute -y

```bash
# 1. 기본 경로 추적
traceroute google.com

# 2. IP 주소로만 보기 (-n) ⭐️ 속도 향상
# 도메인 이름 해석하느라 느려지는 걸 방지함. 결과가 훨씬 빨리 뜸.
traceroute -n google.com 
```

**결과 해석**

| Hop | IP 주소      | 응답 시간(ms)             | 상태    | 해석                           |
| --: | ---------- | --------------------- | ----- | ---------------------------- |
|   1 | 172.18.0.1 | 0.416 / 0.341 / 0.318 | 정상 응답 | **컨테이너의 게이트웨이 (Docker 브리지)** |
|   2 | * * *      | 없음                    | 응답 없음 | 중간 라우터가 ICMP 응답 차단           |
|   3 | * * *      | 없음                    | 응답 없음 | ″                            |
|   4 | * * *      | 없음                    | 응답 없음 | ″                            |
|   … | * * *      | 없음                    | 응답 없음 | ″                            |
|  30 | * * *      | 없음                    | 응답 없음 | 최종 목적지까지 경로는 있으나 응답 차단       |

- **숫자(Hop):** 거쳐간 단계 수.
- **`* * *` (별표):** 해당 구간의 장비가 보안상 응답을 거부했거나(방화벽), 패킷이 유실된 경우.

---
## Code Core Points: ③ `nslookup` vs `dig` (주소록 확인) 

DNS(도메인 네임 시스템)를 조회한다. 
`nslookup`은 구형이고, **`dig`가 더 상세하고 강력해서 실무 표준**이다.

### A. `nslookup` (간단 확인)

```bash
# 도메인의 IP 확인
nslookup google.com 

# 특정 DNS 서버(8.8.8.8)에게 물어보기
nslookup google.com 8.8.8.8 
```


### B. `dig` (상세 확인)

>sudo apt update
>sudo apt install bind9-dnsutils -y

```bash
# 1. 기본 상세 조회
dig google.com

# 2. 결과만 짧게 보기 (+short) ⭐️ 실무용
# 주저리주저리 설명 빼고 IP만 딱 보여줌.
dig +short google.com 

# 3. DNS 경로 추적 (+trace) ⭐️ 트러블슈팅용
# 내 컴퓨터 -> 루트 서버 -> .com 서버 -> 구글 서버로 가는 과정을 다 보여줌.
dig +trace google.com 

# 4. 특정 레코드 조회 (MX, NS 등)
# "지메일 메일 서버(MX)가 어디야?"
dig gmail.com mx 
```

**결과**

```text
dig google.com

;; ANSWER SECTION:
google.com.   300   IN   A   142.251.119.139
```

|항목|의미|
|---|---|
|`google.com.`|조회한 도메인|
|`300`|TTL (초) → 5분 동안 캐시 가능|
|`IN`|인터넷 클래스|
|`A`|A 레코드 (IPv4 주소)|
|`142.251.119.139`|실제 접속 IP|


```text
+trace
.            NS   a.root-servers.net
com.         NS   a.gtld-servers.net
google.com.  NS   ns1.google.com
google.com.  A    142.251.119.139
```

|단계|설명|
|---|---|
|루트 서버 (`.`)|“.com은 어디가 담당?”|
|TLD 서버 (`.com`)|“google.com은 어디가 담당?”|
|권한 서버 (NS)|실제 google DNS|
|최종 A 레코드|google.com IP 반환|

```text
dig gmail.com mx

gmail.com.  300  IN  MX  5 gmail-smtp-in.l.google.com.
gmail.com.  300  IN  MX 10 alt1.gmail-smtp-in.l.google.com.
```

|항목|의미|
|---|---|
|`MX`|메일 서버 레코드|
|`5`, `10`|우선순위 (숫자 낮을수록 우선)|
|`gmail-smtp-in.l.google.com`|실제 메일 수신 서버|

---
## 초보자가 자주 하는 실수 (Misconceptions)

### ① "`ping`은 되는데 웹사이트가 안 열려요!"

- `ping`은 단순히 "길이 뚫려있냐(L3 Network)"만 확인합니다.
- 웹 서버 프로그램(Nginx/Apache)이 죽었거나(L7 Application), 포트(80/443)가 막혔을 수 있습니다. 
- 이때는 `telnet`이나 `curl`로 확인해야 합니다.

### ② "`traceroute` 중간에 `* * *` 뜨면 망한 건가요?"

- 아닙니다. 보안 강한 라우터들은 일부러 `traceroute` 요청을 무시(Drop)하기도 합니다.
- 최종 목적지까지 잘 도착한다면 중간의 별표는 무시해도 됩니다.

### ③ "`nslookup` 쓰면 안 되나요?"

- 써도 됩니다! 하지만 `dig`가 응답 시간(Query Time), 권한 섹션(Authority Section) 등 더 전문적인 정보를 줍니다.
- 엔지니어라면 `dig`를 쓰는 게 좀 더 "전문가스러워" 보입니다. 