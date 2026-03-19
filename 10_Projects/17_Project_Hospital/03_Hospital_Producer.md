---
aliases:
  - Hospital Producer
  - 응급실 Producer
  - er_realtime
tags:
  - Project
related:
  - "[[00_Hospital_Project]]"
  - "[[02_Hospital_Data_Structure]]"
  - "[[Python_ElementTree]]"
  - "[[Python_Requests_Methods]]"
  - "[[Docker_Host_vs_Internal_Network]]"
  - "[[Python_Functions]]"
  - "[[Python_DateTime]]"
  - "[[Kafka_Python_Producer]]"
  - "[[Kafka_Error_Handling_Retry]]"
---


# 03_Hospital_Producer — 응급실 데이터 Kafka 발행

## 이 단계의 목표

```
① Kafka 토픽 생성 (er-realtime)
② API ① + ② + ③ 수집 → hpid 병합 → er-realtime 토픽 발행
③ Producer 정상 실행 확인 (로그 출력)

Spark Consumer / PostgreSQL 적재 → [[04_Hospital_Kafka_Spark]]
```

>Spark Consumer / PostgreSQL 적재 → [[04_Hospital_Kafka_Spark]]

---

---

## 왜 Python 직접 저장이 아니라 Kafka 를 쓰는가?

```
Python 으로 API 호출 → PostgreSQL 직접 INSERT 도 가능하다
그런데 굳이 Kafka 를 중간에 끼우는 이유가 있다
```

```
직접 저장 방식:
  API → Python → PostgreSQL

  문제:
  ① DB가 느려지면 수집도 같이 느려짐
     API 는 5분마다 417개 병원 데이터를 뱉는데
     PostgreSQL INSERT 가 느리면 → 다음 수집 사이클 지연

  ② 소비자가 하나밖에 없음
     Spark 로 실시간 처리 + Superset 시각화 + Airflow 배치
     → 여러 곳에서 같은 데이터를 동시에 써야 하는데 불가능

  ③ 데이터 유실 위험
     DB INSERT 실패 시 → 그 시점 데이터 영구 손실
```

```
Kafka 를 쓰면:
  API → Python(Producer) → Kafka → Spark(Consumer) → PostgreSQL
                                  → Airflow(Consumer) → 배치 분석
                                  → 미래 Consumer 추가 가능

  ① 버퍼 역할
     API 수집 속도와 DB 저장 속도를 분리
     Kafka 에 먼저 쌓아두고 → Consumer 가 자기 속도로 꺼내감

  ② 다중 소비 (Pub/Sub)
     같은 토픽을 Spark / Airflow / 대시보드 여러 곳에서 동시에 읽기 가능
     Producer 는 한 번만 발행 → Consumer 는 각자 읽음

  ③ 메시지 보존
     기본 7일 보존
     Consumer 가 실패해도 다시 읽기 가능 (offset 으로 위치 기억)

  ④ 실시간 파이프라인 경험
     이게 현업에서 실제로 쓰는 구조
     Kafka 없이 만들면 "Python 스크립트" / Kafka 넣으면 "데이터 파이프라인"
```

```
이 프로젝트에서 Kafka 가 필요한 이유:
  5분마다 417개 병원 데이터 → 빠르게 수집 필요
  Spark Streaming 으로 실시간 처리 → Kafka 없이 불가
  Airflow 배치 분석도 같은 데이터 활용
  → 여러 Consumer 가 동시에 쓰는 구조 = Kafka 가 적합
```

---

---

# ① Kafka 토픽 생성

```
⚠️ 토픽 생성 먼저, Producer 실행 나중
토픽 없이 Producer 실행하면 메시지가 쌓이지 않음
```


```bash
# er-realtime 토픽 생성
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic er-realtime --partitions 1 --replication-factor 1

# 토픽 확인
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --list
```

>토픽생성 방법 그 의미 [[Kafka_CLI_Cheatsheet]] 참고 

---

---

# ② /etc/hosts 설정 — 맥북에서 kafka 이름 인식

```
맥북에서 직접 실행 시 "kafka" 라는 이름을 모름
→ NoBrokersAvailable 에러

→ 맥북 /etc/hosts 에 kafka = 127.0.0.1 로 등록
```

```bash
sudo vi /etc/hosts
# 아래 줄 추가
127.0.0.1 kafka
```

```
이후:
  맥북에서 kafka:9093 → 127.0.0.1:9093 로 해석
  컨테이너에서 kafka:9092 → Docker 서비스명으로 해석
  → Producer / Consumer 모두 kafka:9092 로 통일 가능
```


>[[Docker_Host_vs_Internal_Network#⑤ /etc/hosts 설정 — 맥북에서 kafka 이름 인식]] 참고 

---

---

# ② 폴더 구조

```
producer/
├── producer.py       ← 메인 Producer 코드
├── requirements.txt  ← 의존성
└── .env              ← API 키 (프로젝트 루트 .env 공유)
```

```
requirements.txt:
kafka-python==2.0.2
requests==2.31.0
python-dotenv==1.0.0
```

---

---

# ③ Producer 코드 — producer.py

>Python 3.10+ 필요 (`ET.Element | None` 문법) → [[Linux_Python_Env#④ pyenv]] 
>공공 API 성공 코드 `{"00", "0000"}` — API 마다 다름 (홈페이지에는 00이라고 적혀있지만 혹시 몰라 0000 추가 )
>xml보니 Y에도 공백이 많은게 숨어있음 ->strip사용


```python
import os
import json
import time
import requests
import xml.etree.ElementTree as ET
from datetime import datetime
from dotenv import load_dotenv
from urllib.parse import unquote   
from kafka import KafkaProducer
from kafka.errors import KafkaError

load_dotenv()

# ── 설정 ────────────────────────────────────────────────────────
KAFKA_BOOTSTRAP = os.getenv("KAFKA_BOOTSTRAP_SERVERS", "kafka:9092")
API_KEY         = unquote(os.getenv("HOSPITAL_API_KEY", ""))  # 이중 인코딩 방지
BASE_URL        = "http://apis.data.go.kr/B552657/ErmctInfoInqireService"
POLL_INTERVAL   = 300      # 5분 (초)
NUM_OF_ROWS     = "1000"   # 전국 수집
SUCCESS_CODES   = {"00", "0000"}  # 공공 API 성공 코드 일정하지 않음


# ── 유틸 함수 ────────────────────────────────────────────────────
def parse_int(val, default=None) -> int | None:
    """None 또는 빈 문자열을 안전하게 int 변환. 실패 시 None 반환"""
    try:
        return int(val) if val else default
    except (ValueError, TypeError):
        return default


def parse_yn(val: str) -> str:
    """Y/N 값 정규화
    "Y         " → "Y"   (공백 제거)
    "정보미제공"  → ""    (VARCHAR(1) 저장 불가)
    None / ""   → ""
    """
    if not val:
        return ""
    v = val.strip()
    return v if v in ("Y", "N") else ""


def debug_xml(root: ET.Element, max_items: int = 1) -> None:
    """API 응답 XML 구조 확인용 — 개발 단계에서 필드명/값 확인할 때 사용
    API 마다 XML 구조가 다르므로 새 엔드포인트 추가 시 반드시 확인

    사용법:
        root = fetch_xml("새엔드포인트")
        debug_xml(root)     ← 첫 번째 item 필드 전체 출력
    """
    items = root.findall(".//item")
    print(f"[DEBUG] 전체 {len(items)}건")
    for item in items[:max_items]:
        print("[DEBUG] ── item 구조 ──")
        for child in item:
            print(f"  {child.tag}: {repr(child.text)}")


# ── API 호출 ────────────────────────────────────────────────────
def fetch_xml(path: str, extra_params: dict = None) -> ET.Element | None:
    """공통 API 호출 + XML 파싱"""
    if extra_params is None:
        extra_params = {}

    params = {
        "serviceKey": API_KEY,
        "numOfRows":  NUM_OF_ROWS,
        "pageNo":     "1",
        **extra_params
    }
    try:
        r = requests.get(f"{BASE_URL}/{path}", params=params, timeout=10)
        r.raise_for_status()
        root = ET.fromstring(r.content)

        result_code = root.findtext(".//resultCode", "")
        if result_code not in SUCCESS_CODES:
            msg = root.findtext(".//resultMsg", "알 수 없음")
            print(f"[API 에러] {path}: {result_code} - {msg}")
            return None
        return root

    except requests.exceptions.Timeout:
        print(f"[타임아웃] {path}")
        return None
    except requests.exceptions.RequestException as e:
        print(f"[네트워크 에러] {path}: {e}")
        return None
    except ET.ParseError as e:
        print(f"[XML 파싱 에러] {path}: {e}")
        return None


# ── 수집 함수 ────────────────────────────────────────────────────
def fetch_realtime_beds() -> dict:
    """① 응급실 실시간 가용병상 → {hpid: {필드}} 딕셔너리

    수집 항목:
      hvec       응급실 가용병상 수 ← 핵심
      hvoc       수술실 가용 수
      hvctayn    CT 가용 Y/N
      hvventiayn 인공호흡기 가용 Y/N
    """
    root = fetch_xml("getEmrrmRltmUsefulSckbdInfoInqire")
    if root is None:
        return {}

    # debug_xml(root)  ← 필드명 확인 필요 시 주석 해제

    result = {}
    for item in root.findall(".//item"):
        hpid = item.findtext("hpid", "")
        if not hpid:
            continue
        result[hpid] = {
            "hpid":       hpid,
            "hpname":     item.findtext("dutyName", ""),
            "hvec":       parse_int(item.findtext("hvec")),
            "hvoc":       parse_int(item.findtext("hvoc")),
            "hvctayn":    parse_yn(item.findtext("hvctayn")),
            "hvventiayn": parse_yn(item.findtext("hvventiayn")),
        }
    print(f"[가용병상] {len(result)}곳 병원 수집되었습니다.")
    return result


def fetch_messages(hpids: list) -> dict:
    """③ 병원별 메시지 → {hpid: notice_msg} 딕셔너리

    hpid 별로 개별 호출 → 0.1초 간격 (API 과부하 방지)
    429 발생 시 fetch_messages_mock() 으로 교체
    """
    result = {}
    for hpid in hpids:
        root = fetch_xml("getEmrrmSrsillDissMsgInqire", {"hpid": hpid})
        if root is None:
            result[hpid] = ""
            continue
        result[hpid] = root.findtext(".//symBlkMsg", "")
        time.sleep(0.1)
    print(f"[메시지] {len(result)}곳 병원 수집 완료")
    return result


def fetch_messages_mock(hpids: list) -> dict:
    """③ 메시지 API Mocking — 429 발생 시 임시 대체
    실제 배포 시 fetch_messages() 로 교체
    """
    import random
    mock_messages = [
        "응급실 병상 여유 있습니다.",
        "현재 응급실 포화 상태입니다. (중증 환자 외 수용 불가)",
        "CT/MRI 장비 점검 중. 관련 환자 수용 불가.",
        "소아응급환자 정상 수용 가능합니다.",
        "응급수술 진행 중. 외상환자 수용 지연 (대기 2시간 예상)",
        "", "", "",  # 메시지 없는 병원도 있음
    ]
    result = {hpid: random.choice(mock_messages) for hpid in hpids}
    print(f"[메시지 Mock] {len(result)}개 병원 (가짜 데이터)")
    return result


# ── 병합 ────────────────────────────────────────────────────────
def merge_data(beds: dict, messages: dict) -> list:
    """① + ③ → hpid 기준 병합 → 리스트 반환

    ② 중증질환 수용가능정보 제외 이유:
      MKioskTy* 대부분 "정보미제공" → 분석 신뢰도 없음
      인증 기반 역량 정보는 er_hospitals(mk_*) 에서 조회
    """
    merged = []
    now = datetime.now().isoformat()

    for hpid, bed_data in beds.items():
        row = {
            **bed_data,
            "notice_msg": messages.get(hpid, ""),
            "data_type":  "er_realtime",
            "created_at": now,
        }
        merged.append(row)
    return merged


# ── Producer 클래스 ─────────────────────────────────────────────
class HospitalProducer:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=KAFKA_BOOTSTRAP,
            value_serializer=lambda v: json.dumps(
                v, ensure_ascii=False).encode("utf-8"),
            acks="all",
            retries=3,
        )
        print(f"[Hospital Producer] Kafka 연결: {KAFKA_BOOTSTRAP}")

        # 인메모리 캐싱 — 메시지 API 1시간 주기 (트래픽 최적화)
        self.cached_messages     = {}
        self.last_msg_fetch_time = 0
        self.MSG_POLL_INTERVAL   = 3600

    def _send(self, topic: str, data: dict):
        try:
            self.producer.send(topic, value=data)
        except KafkaError as e:
            print(f"[Kafka 발행 에러] {topic}: {e}")

    def collect_and_publish(self):
        current_time = time.time()
        print(f"\n[수집 시작] {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")

        # ① 가용병상
        beds = fetch_realtime_beds()
        if not beds:
            print("[경고] 가용병상 데이터 없음")
            return 0

        # ③ 메시지 (캐싱 — 1시간마다 갱신)
        if current_time - self.last_msg_fetch_time > self.MSG_POLL_INTERVAL:
            print("[메시지 갱신] 1시간 경과. API 새로 호출합니다...")
            self.cached_messages     = fetch_messages(list(beds.keys()))
            # self.cached_messages   = fetch_messages_mock(list(beds.keys()))  # 429 시 교체
            self.last_msg_fetch_time = current_time
        else:
            time_left = int(self.MSG_POLL_INTERVAL - (current_time - self.last_msg_fetch_time))
            print(f"[메시지 캐시] 재사용 (다음 갱신: {time_left // 60}분 후)")

        # 병합 → 발행
        merged = merge_data(beds, self.cached_messages)
        for row in merged:
            self._send("er-realtime", row)

        self.producer.flush()
        print(f"[발행 완료] {len(merged)}건 → er-realtime 토픽")
        return len(merged)

    def run(self):
        print(f"🚑 Hospital Producer 가동 시작 (수집 주기: {POLL_INTERVAL}초)\n")
        while True:
            try:
                count = self.collect_and_publish()
                if count == 0:
                    print("[경고] 발행된 데이터가 없습니다. API 상태를 확인하세요.")
                print(f"[대기] {POLL_INTERVAL}초 후 다음 주기를 시작합니다...\n")
                time.sleep(POLL_INTERVAL)
            except KeyboardInterrupt:
                print("\n[종료] Producer 를 안전하게 중지합니다.")
                self.producer.close()
                break
            except Exception as e:
                print(f"[예외 발생] {e}")
                time.sleep(60)


if __name__ == "__main__":
    hp = HospitalProducer()
    hp.run()
```

---

---

# ④ 실행

```bash
# 맥북에서 직접 실행
cd producer
pip3 install -r requirements.txt
python3 producer.py
```

```
[Hospital Producer] Kafka 연결!!:kafka:9093
🚑 Hospital Producer 가동 시작 (수집 주기: 300초)


[수집 시작] 2026-03-18 19:20:08
[가용병상] 417곳 병원 수집되었습니다.
[중증질환] 133곳 병원 수집되었습니다.
[메시지 갱신] 1시간경과 API 새로 호출합니다.
[병원 메시지] 417개 수집!
[발행완료!]417건 ->er-realtime Topic
[대기]300초 후 다음 주기를 시작합니다!..
```

## 발행 확인 — CLI Consumer

```bash
# 발행된 메시지 JSON 으로 예쁘게 확인
docker exec -i hospital-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic er-realtime \
    --max-messages 3 2>/dev/null \
    | sed 's/}{/}\n{/g' \
    | jq
```

```json
// 실제 출력 결과 (2026-03-18)
{
  "hpid": "A2200008",
  "hpname": "강릉아산병원",
  "hvec": 1,
  "hvoc": 14,
  "hvctayn": "Y",
  "hvventiayn": "Y",
  "notice_msg": "소아응급환자 정상 수용 가능합니다.",
  "data_type": "er_realtime",
  "created_at": "2026-03-18T20:06:32.938259"
}
```

```
✅ STEP 3 완료 기준:
  가용병상 417곳 수집
  발행 완료 417건
  CLI Consumer 에서 JSON 메시지 확인

 region / duty_addr 는 er_hospitals 에서 JOIN 으로 조회 해야할것 같다 
```


---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`NoBrokersAvailable`|kafka 이름 인식 못 함|아래 상세 참고|
|`resultCode != 00`|API 키 오류 또는 서버 에러|API 키 확인 / 잠시 후 재시도|
|발행 0건|API 데이터 없음|`fetch_realtime_beds()` 반환값 확인|
|XML 파싱 에러|API 서버가 HTML 에러 페이지 반환|`r.content.decode()` 로 원본 확인|
|메시지 수집 느림|417개 병원 순차 호출|캐싱 로직으로 1시간 주기 처리|

## NoBrokersAvailable 상세

```
에러 발생 상황:
  KAFKA_BOOTSTRAP_SERVERS=kafka:9092 로 설정 후
  맥북 터미널에서 python3 producer.py 실행

원인:
  맥북은 "kafka" 라는 호스트명을 모름
  Docker 내부에서만 kafka = 컨테이너 IP 로 DNS 해석 가능
  맥북 터미널 = Docker 외부 → kafka 이름 해석 불가 → 연결 실패
```

```
해결 방법 1 — localhost:9093 직접 사용 (가장 간단)
  KAFKA_BOOTSTRAP_SERVERS=localhost:9093

해결 방법 2 — /etc/hosts 등록 후 kafka:9093 사용
  127.0.0.1 kafka 추가 → kafka:9093 으로 접속
  (포트는 반드시 9093, 9092 는 컨테이너 내부 포트)
```

## 발행완료 로그 나왔는데 Consumer 에 메시지 없음 — ADVERTISED_LISTENERS 문제

```
증상:
  python3 producer.py 에서 "[발행완료!] 417건" 로그 나옴
  CLI Consumer 에서는 아무것도 안 나옴

원인:
  Producer → kafka:9093 (bootstrap 연결 성공)
  Kafka 브로커: "나한테 메시지 보낼 때 kafka:9092 로 와" (ADVERTISED_LISTENERS)
  Producer → kafka:9092 시도
  → /etc/hosts 로 kafka = 127.0.0.1
  → kafka:9092 = 127.0.0.1:9092
  → 9092 는 포트포워딩 안 됨 (9093만 열려있음)
  → 실제 메시지 전송 실패
  → KafkaProducer 는 에러 없이 그냥 넘어감 → 발행완료 로그가 거짓말

KAFKA_ADVERTISED_LISTENERS 를 localhost 로 바꾸면 안 되는 이유:
  Spark / Airflow 컨테이너가 "localhost:9092 로 와" 안내를 받음
  컨테이너 안에서 localhost = 자기 자신 → Kafka 아님 → 연결 실패
  → 컨테이너끼리 통신이 깨짐
```


```yaml
# 해결 — docker-compose.yml 에 9092 포트 추가
kafka:
  ports:
    - "9093:9092"   # 기존
    - "9092:9092"   # 추가 ← 맥북에서 9092 직접 접근 가능하게
```


```bash
# 재시작
docker compose down
docker compose up -d

# 토픽 재생성 (down 시 삭제됨)
docker exec -it hospital-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 --create \
    --topic er-realtime --partitions 1 --replication-factor 1

# Producer 재실행
python3 producer.py
```

```
포트 추가 후 KAFKA_BOOTSTRAP_SERVERS=kafka:9092 로 설정 가능:
  kafka:9092 → 127.0.0.1:9092 → 포트포워딩 → 컨테이너 9092 ✅
  ADVERTISED_LISTENERS=kafka:9092 → 컨테이너끼리도 동일 ✅
```


>[[Docker_Host_vs_Internal_Network#⑤ /etc/hosts 설정 — 맥북에서 kafka 이름 인식]] 참고 


----


✅ 완료되면 → [[04_Hospital_Kafka_Spark]] 으로 이동