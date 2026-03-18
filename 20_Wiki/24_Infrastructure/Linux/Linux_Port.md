---
aliases:
  - 포트
  - lsof
  - netstat
  - curl
  - wget
  - 포트 확인
tags:
  - Linux
related:
  - "[[00_Linux_HomePage]]"
  - "[[Linux_Network_Basic]]"
  - "[[Linux_Network_Commands]]"
  - "[[CS_TCP_IP]]"
---

# Linux_Port — 포트 개념 및 명령어

## 한 줄 요약

```
포트 = 컴퓨터 안에서 어떤 프로그램으로 데이터를 보낼지 구분하는 번호
어떤 포트가 열려있는지 확인하고 직접 요청 보내는 도구
```

---

---

# ① 포트 개념

```
IP 주소  → 어느 컴퓨터로 갈지
포트     → 그 컴퓨터 안에서 어떤 프로그램으로 갈지

192.168.0.1:8080
  IP: 192.168.0.1  → 이 컴퓨터로
  Port: 8080       → 8080 포트를 듣고 있는 프로그램으로

비유:
  IP = 건물 주소
  포트 = 건물 안 호실 번호
  192.168.0.1:5432 = A 건물 5432호 (PostgreSQL 사무실)
  192.168.0.1:9092 = A 건물 9092호 (Kafka 사무실)
```

## 잘 알려진 포트 (Well-known Ports)

```
22    SSH          원격 접속
80    HTTP         웹 (비암호화)
443   HTTPS        웹 (암호화)
5432  PostgreSQL   DB
9092  Kafka        메시지 큐
3306  MySQL        DB
6379  Redis        캐시
8080  Spark UI     (프로젝트 관례)
8082  Airflow UI   (프로젝트 관례)
8088  Superset     (프로젝트 관례)
8089  Superset     (hospital 프로젝트)

0~1023    시스템 예약 (root 권한 필요)
1024~65535 일반 사용 가능
```

---

---

# ② lsof — 포트 점유 확인

```
lsof = List Open Files
리눅스에서 "모든 것은 파일" → 소켓(네트워크 연결)도 파일
어떤 프로세스가 어떤 포트를 점유하고 있는지 확인
```

```bash
# 특정 포트 점유 확인
lsof -i :5432
lsof -i :9092
lsof -i :8080

# 출력 예시
COMMAND   PID  USER  FD  TYPE DEVICE SIZE/OFF NODE NAME
postgres  1234 user  5u  IPv4 12345       0t0  TCP *:5432 (LISTEN)
# → postgres 프로세스(PID 1234)가 5432 포트 점유 중

# 포트 비어있으면 아무것도 안 뜸

# TCP 연결 전체 확인
lsof -i TCP

# 특정 프로세스가 열고 있는 포트
lsof -p 1234
```

```
Docker 프로젝트에서 자주 쓰는 상황:
  "port is already allocated" 에러 발생 시
  lsof -i :9092  → 누가 9092 쓰고 있는지 확인
  → train-kafka 컨테이너가 쓰고 있으면
  → hospital 프로젝트는 9093 으로 변경
```

---

---

# ③ netstat -tulnp — 열려있는 포트 전체 확인

```bash
# 열려있는 모든 포트 확인
netstat -tulnp

# 옵션 의미:
# -t  TCP
# -u  UDP
# -l  LISTEN 상태 (대기 중인 것만)
# -n  숫자로 표시 (도메인명 대신 IP)
# -p  프로세스 이름/PID

# 출력 예시
Proto  Local Address    Foreign Address  State   PID/Program
tcp    0.0.0.0:5432     0.0.0.0:*        LISTEN  1234/postgres
tcp    0.0.0.0:9092     0.0.0.0:*        LISTEN  5678/kafka
tcp    0.0.0.0:8080     0.0.0.0:*        LISTEN  9012/spark

# macOS 에서는 netstat -tulnp 대신:
netstat -an | grep LISTEN
lsof -i -P -n | grep LISTEN   ← macOS 권장
```

```bash
# 특정 포트만 필터
netstat -tulnp | grep 5432
netstat -tulnp | grep LISTEN
```

```
State 의미:
  LISTEN       연결 대기 중 (서버)
  ESTABLISHED  연결됨 (통신 중)
  TIME_WAIT    연결 종료 중
  CLOSE_WAIT   상대방이 종료 요청
```

---

---

# ④ curl — HTTP 요청 보내기

```
curl = Client URL
터미널에서 HTTP 요청을 직접 보내는 도구
API 테스트 / 파일 다운로드 / 응답 확인
```

```bash
# 기본 GET 요청
curl https://apis.data.go.kr/...

# 응답을 파일로 저장
curl -o output.xml https://apis.data.go.kr/...

# 헤더 포함 응답 보기
curl -i https://apis.data.go.kr/...

# 자세한 통신 과정 보기 (디버깅)
curl -v https://apis.data.go.kr/...

# POST 요청
curl -X POST https://api.example.com/data \
     -H "Content-Type: application/json" \
     -d '{"key": "value"}'

# 파라미터 포함 GET
curl "https://apis.data.go.kr/endpoint?serviceKey=KEY&STAGE1=서울"

# 응답 상태코드만 확인
curl -o /dev/null -s -w "%{http_code}" https://example.com
```

```
자주 쓰는 옵션:
  -o 파일명    응답을 파일로 저장
  -O           URL 의 파일명으로 저장
  -i           응답 헤더 포함 출력
  -v           상세 통신 과정 (디버깅)
  -s           진행률 숨김 (silent)
  -X METHOD    HTTP 메서드 지정
  -H "헤더"    요청 헤더 추가
  -d "데이터"  요청 바디 데이터
  --max-time N 타임아웃 (초)
```

```bash
# 실전: Superset 서버 올라왔는지 확인
curl -o /dev/null -s -w "%{http_code}" http://localhost:8088
# 200 나오면 정상 / 000 이면 서버 안 뜸

# 실전: Kafka 포트 열렸는지 확인
curl -v telnet://localhost:9092 2>&1 | head -5
```

---

---

# ⑤ wget — 파일 다운로드

```
wget = Web Get
주로 파일 다운로드에 특화
curl 보다 다운로드 기능이 강력
```

```bash
# 기본 다운로드
wget https://example.com/file.zip

# 파일명 지정
wget -O output.zip https://example.com/file.zip

# 백그라운드 다운로드
wget -b https://example.com/largefile.zip

# 재시도 횟수 지정
wget --tries=3 https://example.com/file.zip

# 특정 폴더에 저장
wget -P /tmp https://example.com/file.zip

# 재귀 다운로드 (웹페이지 전체)
wget -r https://example.com
```

---

---

# ⑥ curl vs wget

|항목|curl|wget|
|---|---|---|
|주 용도|API 테스트 / 데이터 전송|파일 다운로드|
|HTTP 메서드|GET/POST/PUT/DELETE 전부|주로 GET|
|응답 출력|표준 출력 (터미널)|파일로 저장|
|재시도|수동 설정|자동 재시도|
|파이프 연결|✅ 자주 씀|제한적|
|설치|macOS 기본 포함|별도 설치 필요|

```bash
# API 응답을 바로 파이썬으로 처리
curl -s "https://api.example.com/data" | python3 -m json.tool

# 다운로드 후 압축 해제
wget https://example.com/file.tar.gz && tar -xzf file.tar.gz
```

---

---

# 실전 패턴 — 포트 트러블슈팅

```bash
# 1. 포트 충돌 에러 발생 시
docker compose up -d
# Error: port is already allocated 9092

# 2. 누가 쓰고 있는지 확인
lsof -i :9092
# → train-kafka 컨테이너

# 3. 서비스 올라왔는지 curl 로 확인
curl -s -o /dev/null -w "%{http_code}" http://localhost:8088
# 200 → Superset 정상
# 000 → 아직 안 뜸

# 4. 포트 전체 현황
lsof -i -P -n | grep LISTEN  # macOS
netstat -tulnp                # Linux
```