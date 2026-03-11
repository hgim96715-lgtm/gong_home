---
aliases:
  - Kafka Producer
  - kafka-python
  - 열차 Producer
  - train-schedule
  - train-realtime
tags:
  - Project
related:
  - "[[00_Seoul Station Real-Time Train Project|train-project]]"
  - "[[Docker_Compose_Setup]]"
  - "[[Python_DateTime]]"
  - "[[Kafka_CLI_Cheatsheet]]"
  - "[[02_API_Producer]]"
  - "[[Kafka_Python_Consumer]]"
  - "[[Kafka_Python_Producer]]"
  - "[[Kafka_Error_Handling_Retry]]"
  - "[[Kafka_Python_Serialization]]"
  - "[[Python_JSON]]"
  - "[[Python_OS_Module]]"
---
---

# 03_Kafka_Producer — 열차 데이터를 Kafka 로 흘리기

## 이 단계의 목표

```
STEP 2 에서 API 로 가져온 열차 데이터를
Kafka 토픽으로 발행(Produce) 하는 것까지

⚠️ 코레일 당일 실시간 API 막힘 → 설계 변경

train-schedule  토픽 → 당일 운행계획 데이터 (하루 1회)
train-realtime  토픽 → 계획 기반 상태 추정값 (60초마다)
train-delay     토픽 → 전날 계획 vs 실제 지연 분석 (하루 1회, 새벽)
```

---

---

# ① Kafka 토픽 준비

> Kafka 컨테이너가 먼저 실행 중이어야 한다. [[Docker_Compose_Setup]] 참고

## 토픽 생성

```bash
# train-schedule 토픽 생성
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create \
    --topic train-schedule \
    --partitions 1 \
    --replication-factor 1

# train-realtime 토픽 생성
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create \
    --topic train-realtime \
    --partitions 1 \
    --replication-factor 1

# train-delay 토픽 생성 (전날 지연 분석)
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create \
    --topic train-delay \
    --partitions 1 \
    --replication-factor 1
```

## 토픽 확인

```bash
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --list
# train-delay
# train-realtime
# train-schedule
```

> 토픽 명령어 전체 → [[Kafka_CLI_Cheatsheet]]

---

---

# ② 설계 구조

```
TrainProducer (클래스)
├── __init__()            KafkaProducer 생성 + TrainInfo 연결
├── _send()               단건 발행 + 에러 처리
├── run_schedule()        당일 운행계획 → train-schedule (하루 1회)
├── run_estimated()       계획 기반 상태 추정 → train-realtime (60초마다)
├── run_delay_analysis()  전날 계획 vs 실제 비교 → train-delay (하루 1회)
└── run()                 메인 루프

흐름:
  API 호출 (TrainInfo)
      ↓
  JSON 직렬화 (json.dumps → bytes)
      ↓
  Kafka 토픽 발행 (KafkaProducer.send)
      ↓
  flush() → 버퍼에 쌓인 메시지 강제 전송
      ↓
  60초 대기 (time.sleep) → 반복
```

---

---

# ③ 환경 설정

## requirements.txt 확인

```
# producer/requirements.txt (STEP 2 에서 이미 추가됨)

requests==2.31.0
python-dotenv==1.0.0
kafka-python==2.0.2    ← 이 단계에서 사용
```

## .env 확인

```bash
# .env
TRAIN_API_KEY=발급받은_인증키
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

---

---

# ④ Kafka Producer 코드 — producer.py

## 파일 위치

```
seoul-train-realtime-project/
└── producer/
    ├── main.py          ← STEP 2 (API 호출)
    ├── producer.py      ← STEP 3 (Kafka 발행) ← 여기
    └── requirements.txt
```

## producer.py

```python
import os
import json
import time
from datetime import datetime, timedelta
from typing import Optional
from dotenv import load_dotenv
from kafka import KafkaProducer
from kafka.errors import KafkaError, KafkaTimeoutError
from main import TrainInfo

load_dotenv()

KAFKA_BOOTSTRAP = os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092")
POLL_INTERVAL   = 60
MY_STATION      = "서울"
MAX_TARGETS     = 5


def estimate_status(plan_dep: str, plan_arr: str) -> dict:
    """당일 실시간 API 대체 — 계획 시각과 현재 시각 비교로 상태 추정"""
    now   = datetime.now()
    today = now.strftime("%Y-%m-%d")
    try:
        dep_dt = datetime.strptime(f"{today} {plan_dep}", "%Y-%m-%d %H:%M")
        arr_dt = datetime.strptime(f"{today} {plan_arr}", "%Y-%m-%d %H:%M")
    except ValueError as e:
        print(f"🚨 시간 변환 실패 | plan_dep: '{plan_dep}' plan_arr: '{plan_arr}' | {e}")
        return {"status": "시각 정보 없음", "progress_pct": 0}

    diff_dep   = (dep_dt - now).total_seconds() / 60
    diff_arr   = (now - arr_dt).total_seconds() / 60
    total_mins = (arr_dt - dep_dt).total_seconds() / 60
    progress   = max(0, min(100, int((now - dep_dt).total_seconds() / 60 / total_mins * 100))) if total_mins > 0 else 0

    if diff_dep > 15:
        mins   = int(diff_dep)
        status = f"출발 {mins // 60}시간 {mins % 60}분 전 (대기중)" if mins >= 60 else f"출발 {mins}분 전 (대기중)"
    elif 0 < diff_dep <= 15:
        status = f"곧 출발 ({int(diff_dep)}분 후) — 탑승 중"
    elif diff_dep <= 0 and diff_arr < 0:
        remaining = int(abs((arr_dt - now).total_seconds() / 60))
        status    = f"운행 중 ({progress}% 진행, 약 {remaining}분 후 도착 예정)"
    else:
        status = f"도착 완료 ({int(diff_arr)}분 전)"

    return {"status": status, "progress_pct": progress}


def calc_delay_min(plan_hm: str, actual_hm: str) -> Optional[int]:
    """계획 vs 실제 시각 차이(분) — 자정 통과 보정 포함"""
    if "--:--" in (plan_hm, actual_hm) or "" in (plan_hm, actual_hm):
        return None
    try:
        p_h, p_m = map(int, plan_hm.split(":"))
        a_h, a_m = map(int, actual_hm.split(":"))
        diff = (a_h * 60 + a_m) - (p_h * 60 + p_m)
        if diff < -720:    # 밤 → 새벽 통과
            diff += 1440
        elif diff > 720:   # 새벽 → 밤 역주행 (드문 케이스)
            diff -= 1440
        return diff
    except ValueError:
        return None


def delay_label(mins: Optional[int]) -> str:
    if mins is None: return "확인불가"
    if mins <= 0:    return f"정시({mins}분)"
    if mins <= 5:    return f"소폭지연(+{mins}분)"
    if mins <= 30:   return f"지연(+{mins}분)"
    return f"대폭지연(+{mins}분)"


class TrainProducer:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=KAFKA_BOOTSTRAP,
            value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
            acks="all",
            retries=3,
        )
        self.train_info       = TrainInfo()
        self.daily_schedule   = []
        self.current_date     = ""
        self.delay_done_today = False

    def _send(self, topic: str, message: dict):
        try:
            future = self.producer.send(topic, value=message)
            future.get(timeout=10)
        except KafkaTimeoutError:
            print(f"  ⚠️  [{topic}] 전송 Timeout")
        except KafkaError as e:
            print(f"  ❌ [{topic}] 발행 실패: {e}")

    def run_schedule(self, run_ymd: str):
        """당일 운행계획 전체 발행 — 하루 1회"""
        print(f"\n[운행계획] {run_ymd} 발행 시작")
        items = self.train_info.get_train_schedule(run_ymd)
        if not items:
            print("[운행계획] 데이터 없음")
            return
        for item in items:
            item["created_at"] = datetime.now().isoformat()
            item["data_type"]  = "schedule"
            self._send("train-schedule", item)
        self.producer.flush()
        self.daily_schedule = items
        print(f"[운행계획] {len(items)}건 완료 ✅")

    def run_estimated(self, target: dict):
        """계획 기반 상태 추정 → train-realtime 발행"""
        now      = datetime.now()
        plan_dep = TrainInfo._format_dt(target.get("trn_plan_dptre_dt", ""), "--:--")
        plan_arr = TrainInfo._format_dt(target.get("trn_plan_arvl_dt", ""), "--:--")
        estimated = estimate_status(plan_dep, plan_arr)

        self._send("train-realtime", {
            "trn_no"       : target.get("trn_no"),
            "mrnt_nm"      : target.get("mrnt_nm", "노선미상"),
            "dptre_stn_nm" : target.get("dptre_stn_nm"),
            "arvl_stn_nm"  : target.get("arvl_stn_nm"),
            "plan_dep"     : plan_dep,
            "plan_arr"     : plan_arr,
            "status"       : estimated["status"],
            "progress_pct" : estimated["progress_pct"],
            "data_type"    : "estimated",
            "created_at"   : now.isoformat(),
        })
        print(
            f"  [KTX {target.get('trn_no')}호] {plan_dep} 출발 | "
            f"{target.get('mrnt_nm','노선미상'):<6} | "
            f"{target.get('dptre_stn_nm'):<4} ➡️  {target.get('arvl_stn_nm'):<4} | "
            f"{estimated['status']}"
        )

    def run_delay_analysis(self):
        """전날 계획 vs 실제 비교 → train-delay 발행 — 하루 1회 (새벽)"""
        yesterday  = (datetime.now() - timedelta(days=1)).strftime("%Y%m%d")
        print(f"\n[지연분석] {yesterday} 분석 시작...")

        plan_items = self.train_info.get_train_schedule(yesterday)
        if not plan_items:
            print("[지연분석] 운행계획 데이터 없음")
            return

        plan_map = {
            i.get("trn_no"): i for i in plan_items
            if i.get("dptre_stn_nm") == MY_STATION
        }

        count = 0
        for trn_no, plan in plan_map.items():
            time.sleep(0.3)   # API Rate Limit 방지
            actual_items = self.train_info.get_train_realtime(yesterday, trn_no)
            if not actual_items:
                continue

            # 출발: 앞에서부터 (시발역)
            # 도착: 뒤에서부터 역탐색 (종착역 — 중간 기착지 오인 방지)
            actual_dep_item = next((i for i in actual_items if i.get("trn_dptre_dt")), None)
            actual_arr_item = next((i for i in reversed(actual_items) if i.get("trn_arvl_dt")), None)

            plan_dep   = TrainInfo._format_dt(plan.get("trn_plan_dptre_dt", ""), "--:--")
            plan_arr   = TrainInfo._format_dt(plan.get("trn_plan_arvl_dt", ""),  "--:--")
            actual_dep = TrainInfo._format_dt(actual_dep_item.get("trn_dptre_dt", "") if actual_dep_item else "", "--:--")
            actual_arr = TrainInfo._format_dt(actual_arr_item.get("trn_arvl_dt", "") if actual_arr_item else "", "--:--")

            dep_delay = calc_delay_min(plan_dep, actual_dep)
            arr_delay = calc_delay_min(plan_arr, actual_arr)

            self._send("train-delay", {
                "run_ymd"     : yesterday,
                "trn_no"      : trn_no,
                "mrnt_nm"     : plan.get("mrnt_nm", ""),
                "dptre_stn_nm": plan.get("dptre_stn_nm", ""),
                "arvl_stn_nm" : plan.get("arvl_stn_nm", ""),
                "plan_dep"    : plan_dep,
                "plan_arr"    : plan_arr,
                "real_dep"    : actual_dep,
                "real_arr"    : actual_arr,
                "dep_delay"   : dep_delay,
                "arr_delay"   : arr_delay,
                "dep_status"  : delay_label(dep_delay),
                "arr_status"  : delay_label(arr_delay),
                "data_type"   : "delay_analysis",
                "created_at"  : datetime.now().isoformat(),
            })
            print(
                f"  {trn_no}호 | "
                f"출발 계획:{plan_dep} 실제:{actual_dep} [{delay_label(dep_delay)}] | "
                f"도착 계획:{plan_arr} 실제:{actual_arr} [{delay_label(arr_delay)}]"
            )
            count += 1

        self.producer.flush()
        print(f"[지연분석] {count}건 완료 ✅")
        self.delay_done_today = True

    def run(self):
        print(f"🚄 Train Producer 시작 (간격: {POLL_INTERVAL}초)\n")

        self.current_date = datetime.now().strftime("%Y%m%d")
        self.run_schedule(self.current_date)
        self.run_delay_analysis()   # 시작 시 전날 지연 분석

        while True:
            now        = datetime.now()
            today_str  = now.strftime("%Y%m%d")
            current_hm = now.strftime("%H:%M")

            # 날짜 바뀌면 운행계획 갱신
            if today_str != self.current_date:
                print(f"\n📅 날짜 변경 [{self.current_date} → {today_str}]")
                self.current_date     = today_str
                self.delay_done_today = False
                self.run_schedule(self.current_date)

            # 새벽 1~3시 지연 분석 (하루 1회)
            if not self.delay_done_today and "01:00" <= current_hm <= "03:00":
                self.run_delay_analysis()

            print(f"\n{'=' * 58}")
            print(f"  {now.strftime('%Y-%m-%d %H:%M:%S')}  서울역 출발 열차 현황")
            print(f"{'=' * 58}")

            # 방금 출발한 기차(15분 전)부터 표시
            past_15 = (now - timedelta(minutes=15)).strftime("%H:%M")
            targets = [
                item for item in self.daily_schedule
                if item.get("dptre_stn_nm") == MY_STATION
                and TrainInfo._format_dt(item.get("trn_plan_dptre_dt", ""), "99:99") >= past_15
            ][:MAX_TARGETS]

            if targets:
                for target in targets:
                    self.run_estimated(target)
                    time.sleep(0.5)
            else:
                print(f"  {current_hm} 현재 — 금일 서울역 출발 열차 운행 종료 🌝")

            print(f"\n  ⏳ {POLL_INTERVAL}초 후 갱신...\n")
            time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    tp = None
    try:
        tp = TrainProducer()
        tp.run()
    except ValueError as e:
        print(f"설정 오류: {e}")
    except KeyboardInterrupt:
        print("\n🛑 사용자 요청으로 Producer 종료")
    finally:
        if tp:
            tp.producer.flush()
            tp.producer.close()
```


# ⑤ KafkaProducer 핵심 개념

## value_serializer — 직렬화

```python
value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8")

# Kafka 는 bytes 만 전송할 수 있음
# Python dict → 직렬화 → bytes 로 변환해야 함

json.dumps(v, ensure_ascii=False)
#  ensure_ascii=False: 한글을 \uXXXX 로 변환하지 않고 그대로 유지
#  → "서울" 이 "\uc11c\uc6b8" 로 깨지지 않음

.encode("utf-8")
#  문자열 → bytes 변환

# Consumer 에서는 반대로 bytes → decode → json.loads 로 복원
```

## acks — 메시지 전송 보장 수준

```
acks=0    : 서버 응답 안 기다림. 가장 빠르지만 유실 위험
acks=1    : 리더 파티션에 저장되면 성공 (기본값)
acks="all": 모든 replica 에 저장되면 성공. 가장 안전

데이터 파이프라인에서는 acks="all" + retries 조합 권장
```

## send() vs flush()

```python
self.producer.send(topic, value=message)
# 내부 버퍼에 쌓아둠 (비동기)
# 바로 전송되는 게 아님

self.producer.flush()
# 버퍼에 남아있는 메시지 전부 전송 완료될 때까지 대기
# 루프 한 바퀴가 끝난 후에 반드시 호출해야 함
# 안 하면 프로그램 종료 시 버퍼 메시지 유실 가능
```

## future.get() — 동기 확인

```python
future = self.producer.send(topic, value=message)
record_metadata = future.get(timeout=10)
# 전송이 완료될 때까지 최대 10초 대기
# partition, offset 확인 가능 (디버깅 시 유용)

# 고성능이 필요하면 future.get() 제거 (완전 비동기)
# 단, 에러 발생 여부를 즉시 알 수 없음
```

---

---

# ⑥ 실행

## 실행 순서

```bash
# 1. Kafka 컨테이너 실행 확인
docker ps | grep kafka

# 2. 토픽 생성 (처음 한 번만)
docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create --topic train-schedule \
    --partitions 1 --replication-factor 1

docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create --topic train-realtime \
    --partitions 1 --replication-factor 1

docker exec train-kafka \
    /opt/kafka/bin/kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --create --topic train-delay \
    --partitions 1 --replication-factor 1

# 3. Producer 실행
cd producer
python producer.py
```

## 메시지 확인 (Consumer 로 직접 확인)

```bash
# train-schedule 토픽 메시지 확인
docker exec train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-schedule \
    --from-beginning

# train-realtime 토픽 메시지 확인
docker exec train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-realtime \
    --from-beginning

# train-delay 토픽 메시지 확인 (전날 지연 분석)
docker exec train-kafka \
    /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic train-delay \
    --from-beginning
```

## 실행 결과

```
🚄 Train Producer 시작 (간격: 60초)

[운행계획] 20260310 발행 시작
  ✅ [train-schedule] partition=0 offset=0
  ...
[운행계획] 350건 완료 ✅

[지연분석] 20260309 데이터 분석 시작...
  📊 00051호 | 출발 계획:09:00 실제:09:02 [소폭지연 (+2분)]
  📊 00053호 | 출발 계획:11:00 실제:11:00 [정시 (0분)]
  ...
[지연분석] 85건 완료 ✅

==========================================================
  🕐 2026-03-10 14:23:05  서울역 출발 열차 현황
==========================================================
  🚆 [14:30 출발] 00051호 경부선   | 서울 ➡️  부산 | 출발 7분 전 (대기)
  🚆 [15:00 출발] 00055호 경부선   | 서울 ➡️  부산 | 출발 37분 전 (대기)

  ⏳ 60초 후 갱신...
```

---

---

# 트러블슈팅

|증상|원인|해결|
|---|---|---|
|`NoBrokersAvailable`|Kafka 컨테이너 미실행 또는 포트 불일치|`docker ps` 확인, `KAFKA_BOOTSTRAP_SERVERS` 확인|
|`KafkaTimeoutError`|토픽이 없음|토픽 먼저 생성|
|한글 깨짐|`ensure_ascii=True` (기본값)|`ensure_ascii=False` 추가|
|메시지 유실|`flush()` 미호출|루프 끝에 `producer.flush()` 추가|
|발행은 됐는데 Consumer 에서 안 보임|`--from-beginning` 누락|Consumer 실행 시 `--from-beginning` 추가|

---

## 실제 겪은 트러블슈팅

### 1. 실시간 API 가 빈 리스트 반환

```
증상: 오늘 출발 열차의 실시간 정보를 요청하면 항상 [] 반환
원인: 코레일 API 는 당일 실시간 데이터 제공 안 함
      '실시간'이라는 이름과 달리 D-1 배치 데이터였음
```

```
해결: 파이프라인 투트랙 전략으로 전면 재설계
  당일 → 운행계획 + 현재 시각 비교로 상태 추정 (estimate_status)
  전날 → 실제 운행정보로 지연 분석 (run_delay_analysis)
```

---

### 2. 무한 루프 안에서 운행계획 매번 재호출

```
증상: while True 루프마다 get_train_schedule() 호출 → API 서버 부하
원인: 운행계획은 하루에 한 번만 바뀌는 고정 데이터인데 매번 새로 요청함
      공공데이터 서버 IP 차단(Rate Limit) 위험
```

```
해결:
  run_schedule() 로 하루 1회만 호출 → self.daily_schedule 에 저장
  루프에서는 저장된 데이터 재사용
  지연 분석 시 time.sleep(0.3) 추가해서 서버 부하 완화
```

---

### 3. 전광판에 "시각 정보 없음" 대량 출력

```python
# ❌ 문제 코드
datetime.strptime(f"{today}{plan_dep}", "%Y-%m-%d %H:%M")
#                        ↑ 공백 없음

# ✅ 해결 코드
datetime.strptime(f"{today} {plan_dep}", "%Y-%m-%d %H:%M")
#                        ↑ 공백 추가
```

```
원인: 날짜와 시각 사이 공백 누락 → strptime 포맷 불일치 → ValueError
해결: try-except 에 print(e) 찍어서 원인 확인 후 공백 추가
```

---

### 4. 자정 통과 시 지연 시간 음수 계산

```
증상: 23:50 출발 예정 → 00:10 실제 출발 → 계산 결과 -1420분
원인: 날짜 바뀌는 것을 모르고 단순 빼기 → 10 - 1430 = -1420

1440 = 하루 24시간을 분으로 환산 (24 × 60)
 720 = 반나절 12시간을 분으로 환산 (12 × 60)
```

```python
diff = (a_h * 60 + a_m) - (p_h * 60 + p_m)

if diff < -720:     # 밤 → 새벽 통과: -1420 + 1440 = +20 (20분 지연)
    diff += 1440
elif diff > 720:    # 새벽 → 밤 역주행 (드문 케이스): 1430 - 1440 = -10 (10분 조기)
    diff -= 1440
```

---

### 5. Producer 데이터와 DB 스키마 불일치

```
증상: Python 에서 보내는 키와 DB 컬럼명이 달라서 Spark 적재 시 오류 예상
원인: 코드 수정하면서 data_type, progress_pct 같은 새 키 추가했는데
      init.sql 은 예전 API 원본 필드 기준 그대로였음
해결: Python 이 보내는 JSON 구조와 1:1 매칭되도록 테이블 스키마 재설계
      train_realtime, train_delay 테이블 전면 수정
```

---

### 6. 파이썬 버전 관련 문법 에러

```python
# ❌ Python 3.9 이하에서 에러
def calc_delay_min(plan_hm: str, actual_hm: str) -> int | None:

# ✅ 해결 — Optional 사용
from typing import Optional
def calc_delay_min(plan_hm: str, actual_hm: str) -> Optional[int]:
```

```python
# ❌ _format_dt 에 None 전달 시 타입 경고
actual_dep = TrainInfo._format_dt(None, "--:--")

# ✅ 빈 문자열로 대체
actual_dep = TrainInfo._format_dt("", "--:--")
```

---

### 7. 도착 시각 탐색 시 중간 기착지를 종착역으로 오인

```
증상: 부산행 KTX 의 도착 시각이 대전역 도착 시각으로 잘못 계산됨
원인: actual_items 를 앞에서부터 순서대로 탐색하면
      첫 번째 trn_arvl_dt 가 중간 기착지(대전) 도착 시각임
```

```python
# ❌ 앞에서부터 탐색 → 중간 기착지 도착 시각 반환
actual_arr_item = next((i for i in actual_items if i.get("trn_arvl_dt")), None)

# ✅ 뒤에서부터 역탐색 → 종착역 도착 시각 반환
actual_arr_item = next((i for i in reversed(actual_items) if i.get("trn_arvl_dt")), None)
```

---

✅ 완료되면 → [[04_Spark_Streaming]] 으로 이동