---
aliases:
  - netstat
  - lsof
  - ss
  - 포트확인
  - 네트워크상태
  - LISTEN
tags:
  - Linux
related:
  - "[[Time_Synchronization]]"
  - "[[00_Linux_HomePage]]"
  - "[[Process_Management]]"
---
## 개념 한 줄 요약

**"누가(Process) 어느 문(Port)을 열고, 누구와(IP) 대화하고 있는지 감시하는 CCTV."**

* **`netstat`:** 네트워크 연결 상태, 라우팅 테이블, 인터페이스 통계를 보여주는 전통적인 도구입니다
* **`lsof` (List Open Files):** "특정 포트를 누가 잡고 있어?"를 찾아내는 탐정 도구입니다.
* **`ss`:** `netstat`보다 더 빠르고 강력한 최신 대체 명령어입니다. (요즘 리눅스 기본값)

---
## 왜 필요한가? (Why)

**문제점:**
- "웹 서버(Nginx)를 켰는데, **'Address already in use'** 에러가 나면서 죽어버려요."
- "DB에 접속이 안 되는데, 서버가 꺼진 건지 방화벽 문제인지 모르겠어요."

**해결책:**
- **`netstat`** 으로 80번 포트가 **LISTEN(영업 중)** 상태인지 확인합니다
- 만약 이미 누군가 쓰고 있다면 **`lsof`** 로 범인(PID)을 찾아내 **`kill`** 합니다.

---
## Code Core Points: ① 만능 치트키 `netstat` 

옵션이 많지만, 실무에서는 딱 **이 조합** 하나만 기억하면 됩니다.

###  국민 명령어: `netstat -nltp`

(또는 `-tulnp`) 

```bash
# 옵션 해석 (암기 필수!)
# -n (numeric): 도메인 이름 대신 IP 숫자(IP:Port)로 보여줘. (속도 빠름) 
# -l (listening): 현재 문 열고 손님 기다리는(LISTEN) 포트만 보여줘. 
# -t (tcp): TCP 프로토콜만 보여줘. 
# -p (program): 그 포트를 쓰고 있는 프로그램 이름(PID)도 알려줘.
# (-u: UDP도 보고 싶으면 추가)

sudo netstat -nltp
```

#### 결과 해석 (Output Analysis)

| **항목**              | **설명**           | **예시**                                             |
| ------------------- | ---------------- | -------------------------------------------------- |
| **Proto**           | 프로토콜 (TCP/UDP)   | `tcp`                                              |
| **Local Address**   | **누가(서버)** 열고 있나 | `0.0.0.0:80` (전체 허용)<br><br>`127.0.0.1:3306` (내부만) |
| **Foreign Address** | **누구와(외부)** 연결됐나 | `0.0.0.0:*` (아직 연결 없음)                             |
| **State**           | 현재 상태            | `LISTEN` (영업 중)<br><br>`ESTABLISHED` (통화 중)        |
| **PID/Program**     | **범인(프로세스)**     | `1234/nginx`                                       |

>**꿀팁:** `Recv-Q`나 `Send-Q` 숫자가 0이 아니고 계속 쌓여 있다면, 서버가 처리를 못하고 데이터가 밀려있다는 뜻입니다 (과부하 징조).

>**LISTEN**: '가게 문 열고 손님 기다림' (서버 정상 가동 중)
>**ESTABLISHED**: '손님 들어와서 대화 중' (실제 데이터 오가는 중)
>서버 켰는데 접속 안 되면 무조건 `netstat -nltp`부터 쳐봐.
>**LISTEN** 목록에 내 프로그램이 없다? 그럼 아예 켜지지도 않은 거니까 로그부터 봐야 해!"

---
## Code Core Points: ② 범인 색출 `lsof` 

`netstat`은 전체 현황판이라면, `lsof`는 **"특정 포트 범인 잡기"** 에 특화되어 있습니다.

```bash
# 문법: lsof -i :포트번호
# "야, 8080 포트 쓰고 있는 놈 나와."
sudo lsof -i :8080

# 결과 예시:
# COMMAND   PID   USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
# java     3452   root   56u  IPv6   12345      0t0  TCP *:8080 (LISTEN)
```

>**해결:** 범인의 PID(3452)를 확인했으니, `kill -9 3452`로 죽이면 됩니다.
>kill에 대한 내용은 [[Process_Management#Code Core Points ② 처단하기 (`kill`)|kill이란?]]

---
## Code Core Points: ③ 최신 트렌드 `ss`️

최신 리눅스(CentOS 7+, Ubuntu 18+)에서는 `netstat`이 기본 설치가 안 되어 있을 수 있습니다. 
그때 당황하지 말고 **`ss`** 를 쓰세요. 옵션은 `netstat`과 거의 똑같습니다.

```bash
# netstat -nltp 와 똑같은 명령어
sudo ss -nltp
```

---
## 트러블슈팅 (Troubleshooting)

### Q. `netstat` 명령어가 없대요! (`command not found`)

- 최신 배포판에서는 `net-tools` 패키지가 빠져 있어서 그렇습니다.
- **해결 1:** `sudo apt install net-tools` 로 설치.
- **해결 2:** 그냥 최신 명령어인 `ss` 사용 권장.

### Q. `Local Address`가 `127.0.0.1` vs `0.0.0.0` 차이가 뭐죠?

- **`127.0.0.1:80`**: "내 컴퓨터 안에서만 접속 가능해." (보안상 외부 접속 막을 때)
- **`0.0.0.0:80`**: "전 세계 어디서든 나한테 접속해도 돼." (일반적인 웹 서버)

