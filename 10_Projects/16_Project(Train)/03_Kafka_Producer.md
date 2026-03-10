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

# 환경 변수 및 기본 설정
KAFKA_BOOTSTRAP = os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092")
POLL_INTERVAL = 60      # 갱신 주기 (초)
MY_STATION = "서울"       # 전광판 타겟 출발역
MAX_TARGETS = 5         # 한 번에 전광판에 표시할 최대 기차 수


# =====================================================================
# 🛠️ 유틸리티 함수: 시간 계산 및 상태 추정
# =====================================================================

def estimate_status(plan_dep: str, plan_arr: str) -> dict:
    """당일 실시간 데이터 부재를 보완하기 위한 시뮬레이션 전광판 상태 계산"""
    now = datetime.now()
    today = now.strftime("%Y-%m-%d")
    
    try:
        # 시간 문자열 포맷팅 (공백 주의)
        dep_dt = datetime.strptime(f"{today} {plan_dep}", "%Y-%m-%d %H:%M")
        arr_dt = datetime.strptime(f"{today} {plan_arr}", "%Y-%m-%d %H:%M")
    except ValueError as e:
        print(f" 🚨 [디버깅] 시간 변환 실패! 입력값 👉 plan_dep: '{plan_dep}', plan_arr: '{plan_arr}' | 에러 원인: {e}")
        return {"status": "시각 정보 없음", "progress_pct": 0}

    # 분 단위 시간 차이 계산
    diff_dep = (dep_dt - now).total_seconds() / 60
    diff_arr = (now - arr_dt).total_seconds() / 60
    total_mins = (arr_dt - dep_dt).total_seconds() / 60
    
    # 진행률(%) 계산 및 0~100 사이 고정(Clamping)
    progress = max(0, min(100, int((now - dep_dt).total_seconds() / 60 / total_mins * 100))) if total_mins > 0 else 0

    if diff_dep > 15:
        mins = int(diff_dep)
        time_str = f"{mins // 60}시간 {mins % 60}분" if mins >= 60 else f"{mins}분"
        status = f"출발 {time_str}전 (대기중)"
    elif 0 < diff_dep <= 15:
        status = f"곧 출발 ({int(diff_dep)}분 후)- 탑승 중"
    elif diff_dep <= 0 and diff_arr > 0:
        # 실시간 운행정보가 없기 때문에 정시 운행으로 간주하여 시뮬레이션
        remaining = int(diff_arr)
        status = f"운행 중 ({progress}%진행 , 약 {remaining}분 후 도착예정)"
    else:
        status = f"도착 완료 ({int(abs(diff_arr))}분 전)"
        
    return {"status": status, "progress_pct": progress}


def calc_delay_min(plan_hm: str, actual_hm: str) -> Optional[int]:
    """계획 시각 vs 실제 시각을 비교하여 지연 시간(분) 계산 (자정 통과 보정 포함)"""
    if "--:--" in (plan_hm, actual_hm) or "" in (plan_hm, actual_hm):
        return None
    try:
        p_h, p_m = map(int, plan_hm.split(':'))
        a_h, a_m = map(int, actual_hm.split(':'))
        diff = (a_h * 60 + a_m) - (p_h * 60 + p_m)
        
        # 자정을 넘어가는 심야 기차 시간 보정
        if diff < -720: 
            diff += 1440
        elif diff > 730: 
            diff -= 1440
        return diff
    except ValueError:
        return None


def delay_label(mins: Optional[int]) -> str:
    """지연 시간(분)을 사람이 읽기 좋은 텍스트 라벨로 변환"""
    if mins is None: return "확인불가"
    if mins <= 0: return f"정시({mins}분)"
    if mins <= 5: return f"소폭지연(+{mins}분)"
    if mins <= 30: return f"지연(+{mins}분)"
    return f"대폭지연 (+{mins}분)"


# =====================================================================
# 🚄 메인 프로듀서 클래스
# =====================================================================

class TrainProducer:
    def __init__(self):
        # Kafka Producer 초기화 및 JSON 직렬화 설정
        self.producer = KafkaProducer(
            bootstrap_servers=KAFKA_BOOTSTRAP,
            value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode('utf-8'),
            acks="all",
            retries=3,
        )
        self.train_info = TrainInfo()
        self.daily_schedule = []
        self.current_date = ""
        self.delay_done_today = False

    def _send(self, topic: str, message: dict):
        """지정된 토픽으로 단건 메시지 전송"""
        try:
            future = self.producer.send(topic, value=message)
            future.get(timeout=10)
        except KafkaTimeoutError:
            print(f"[{topic}] 전송 Timeout!!")
        except KafkaError as e:
            print(f"[{topic}] 발행 실패: {e}")

    def run_schedule(self, run_ymd: str):
        """하루 1회 당일 운행계획(Schedule)을 적재"""
        print(f"\n[운행계획] {run_ymd} 발행 시작 (하루 1회)")
        items = self.train_info.get_train_schedule(run_ymd)
        
        if not items:
            print("[운행계획] 데이터가 없습니다")
            return
            
        for item in items:
            item["created_at"] = datetime.now().isoformat()
            item["data_type"] = "schedule"
            self._send("train-schedule", item)
            
        self.producer.flush()
        self.daily_schedule = items
        print(f"[운행계획] -> {len(self.daily_schedule)}건 발행완료!")

    def run_estimated(self, target: dict):
        """실시간 API 대용으로 전광판 상태를 추정하여 전송"""
        now = datetime.now()
        trn_no = target.get("trn_no", "?")
        dep_name = target.get("dptre_stn_nm", "?")
        arr_name = target.get("arvl_stn_nm", "?")
        mrnt_nm = target.get("mrnt_nm", "노선미상")
        
        plan_dep = TrainInfo._format_dt(target.get("trn_plan_dptre_dt", ""), "--:--")
        plan_arr = TrainInfo._format_dt(target.get("trn_plan_arvl_dt", ""), "--:--")
        
        estimated = estimate_status(plan_dep, plan_arr)
        
        self._send("train-realtime", {
            "trn_no": trn_no,
            "mrnt_nm": mrnt_nm,
            "dptre_stn_nm": dep_name,
            "arvl_stn_nm": arr_name,
            "plan_dep": plan_dep,
            "plan_arr": plan_arr,
            "status": estimated["status"],
            "progress_pct": estimated["progress_pct"],
            "data_type": "estimated",
            "created_at": now.isoformat()
        })
        
        # 전광판 터미널 출력
        print(
            f"[KTX {trn_no}호 열차] {plan_dep}출발 | {mrnt_nm:<6} |"
            f"{dep_name:<4} ➡️ {arr_name:<4} | {estimated['status']}"
        )

    def run_delay_analysis(self):
        """전날 실제 데이터를 바탕으로 지연 시간을 분석 및 적재"""
        yesterday = (datetime.now() - timedelta(days=1)).strftime("%Y%m%d")
        print(f"\n[지연분석] {yesterday} 분석 시작!")
        
        plan_items = self.train_info.get_train_schedule(yesterday)
        if not plan_items:
            print(f"[지연분석] 운행계획 데이터 없음")
            return
            
        plan_map = {
            item.get("trn_no"): item 
            for item in plan_items 
            if item.get("dptre_stn_nm") == MY_STATION
        }
        
        count = 0
        for trn_no, plan in plan_map.items():
            # API 서버 차단(Rate Limit) 방지용 딜레이
            time.sleep(0.3)
            actual_items = self.train_info.get_train_realtime(yesterday, trn_no)
            
            if not actual_items:
                continue
                
            # 출발역은 순방향, 종착역은 맨 뒤에서부터 역탐색(reversed)
            actual_dep_item = next((i for i in actual_items if i.get("trn_dptre_dt")), None)
            actual_arr_item = next((i for i in reversed(actual_items) if i.get("trn_arvl_dt")), None)
            
            plan_dep = TrainInfo._format_dt(plan.get("trn_plan_dptre_dt", ""), "--:--")
            plan_arr = TrainInfo._format_dt(plan.get("trn_plan_arvl_dt", ""), "--:--")
            actual_dep = TrainInfo._format_dt(actual_dep_item.get("trn_dptre_dt", "") if actual_dep_item else "", "--:--")
            actual_arr = TrainInfo._format_dt(actual_arr_item.get("trn_arvl_dt", "") if actual_arr_item else "", "--:--")
            
            dep_delay = calc_delay_min(plan_dep, actual_dep)
            arr_delay = calc_delay_min(plan_arr, actual_arr)
            
            self._send("train-delay", {
                "run_ymd": yesterday,
                "trn_no": trn_no,
                "mrnt_nm": plan.get("mrnt_nm", ""),
                "dptre_stn_nm": plan.get("dptre_stn_nm", ""),
                "arvl_stn_nm": plan.get("arvl_stn_nm", ""),
                "plan_dep": plan_dep,
                "plan_arr": plan_arr,
                "real_dep": actual_dep,
                "real_arr": actual_arr,
                "dep_delay": dep_delay,
                "arr_delay": arr_delay,
                "dep_status": delay_label(dep_delay),
                "arr_status": delay_label(arr_delay),
                "data_type": "delay_analysis",
                "created_at": datetime.now().isoformat()
            })
            
            print(
                f"{trn_no}호 |"
                f"[출발] : 계획 {plan_dep} -> 실제 {actual_dep} [{delay_label(dep_delay)}] |"
                f"[도착] : 계획 {plan_arr} -> 실제 {actual_arr} [{delay_label(arr_delay)}]"
            )
            count += 1
            
        self.producer.flush()
        print(f"[지연분석] {count}건 발행완료")
        self.delay_done_today = True

    def run(self):
        """메인 무한 루프: 파이프라인의 심장"""
        print(f"Train Producer 시작 (폴링간격: {POLL_INTERVAL}초)\n")
        
        self.current_date = datetime.now().strftime("%Y%m%d")
        self.run_schedule(self.current_date)
        
        # 시작 시 지연 분석 1회 수행
        if not self.delay_done_today:
            self.run_delay_analysis()
            
        while True:
            now = datetime.now()
            today_str = now.strftime("%Y%m%d")
            current_hm = now.strftime("%H:%M")
            
            # 자정 통과 시 스케줄 갱신 로직
            if today_str != self.current_date:
                print(f"\n 하루가 지났군요! 날짜 변경하겠습니다. [{self.current_date} -> {today_str}]")
                self.current_date = today_str
                self.delay_done_today = False
                self.run_schedule(self.current_date)
                
            # 새벽 시간대(01:00 ~ 03:00) 지연 분석 배치 실행
            if not self.delay_done_today and "01:00" <= current_hm <= "03:00":
                self.run_delay_analysis()
                
            # 전광판 출력 헤더
            print(f"\n{'=' * 58}")
            print(f"{now.strftime('%Y-%m-%d %H:%M:%S')} 서울역 기준 출발 열차 현황")
            print(f"\n{'=' * 58}")
            
            # 방금 출발한(15분 전) 기차부터 표시하기 위한 타겟 필터링
            past_15_mins = (now - timedelta(minutes=15)).strftime("%H:%M")
            targets = []
            
            for item in self.daily_schedule:
                dep_time = TrainInfo._format_dt(item.get("trn_plan_dptre_dt", ""), "99:99")
                if item.get("dptre_stn_nm") == MY_STATION and dep_time >= past_15_mins:
                    targets.append(item)
                if len(targets) >= MAX_TARGETS:
                    break
                    
            if targets:
                for target in targets:
                    self.run_estimated(target)
                    time.sleep(0.5)
            else:
                print(f"{current_hm} 현재 - 금일 서울역 출발 열차 운행 종료되었습니다. 편안한 밤 보내세요 🌝")
                
            print(f"\n {POLL_INTERVAL}초 후 갱신..\n")
            time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    tp = None
    try:
        tp = TrainProducer()
        tp.run()
    except ValueError as e:
        print(f"설정오류: {e}")
    except KeyboardInterrupt:
        print("\n 사용자 요청으로 Producer 종료")
    finally:
        # 안전한 자원 반납
        if tp:
            tp.producer.flush()
            tp.producer.close()
```

---

---

# ⑤ KafkaProducer 핵심 개념

## value_serializer — 직렬화

```python
value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8")
```

```
Kafka 는 bytes 만 전송할 수 있음
Python dict → 직렬화 → bytes 로 변환해야 함

json.dumps(v, ensure_ascii=False)
  ensure_ascii=False: 한글을 \uXXXX 로 변환하지 않고 그대로 유지
  → "서울" 이 "\uc11c\uc6b8" 로 깨지지 않음

.encode("utf-8")
  문자열 → bytes 변환

Consumer 에서는 반대로 bytes → decode → json.loads 로 복원
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

## 자정 통과 

**`1440`**: 하루 24시간을 분으로 환산한 값입니다. ($24 \times 60 = 1440$분)
**`720`**: 반나절(12시간)을 분으로 환산한 값입니다. ($12 \times 60 = 720$분)

**밤 11시 50분 기차가 지연되어서 다음 날 새벽 12시 10분에 출발했다면?** 여기서 대참사가 발생합니다.

- 계획: 23:50 (하루 중 **1430번째** 분)
- 실제: 00:10 (하루 중 **10번째** 분)
- 컴퓨터의 무식한 계산 (`diff = a - p`): $10 - 1430 = \textbf{-1420}$

`if diff < -720: diff += 1440` (밤에서 새벽으로 넘어갈 때)
- **해결:** 착각한 만큼 하루 전체 분(`1440`)을 더해줍니다.
- **결과:** $-1420 + 1440 = \textbf{+20}$ (정확히 20분 지연으로 계산 성공!)

`elif diff > 720: diff -= 1440` (새벽에서 밤으로 역주행할 때)
- **의미:** (드문 경우지만) 새벽 00:05 출발 예정 기차가 조금 일찍 출발해서 전날 23:55에 떠났을 때 발생합니다. 컴퓨터는 $1435 - 5 = \textbf{+1430}$(1430분 지연)이라고 착각합니다.
- **해결:** 이번엔 하루 전체 분(`1440`)을 빼줍니다.
- **결과:** $1430 - 1440 = \textbf{-10}$ (정확히 10분 조기 출발로 계산 성공!)



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
python3 producer.py
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
Train Producer 시작 (폴링간격 :60초)

[운행게획] 20260310발행시작(하루1회)

[운행계획]-> 724건 발행완료!

00071호 |[출발] : 계획 21:58 -> 실제 21:58 [정시(0분)] |[도착] : 계획 00:43 -> 실제 00:46 [소폭지연(+3분)]
⚡️ 정보 1페이지 데이터를 가져오는 중입니다...
00073호 |[출발] : 계획 22:28 -> 실제 22:29 [소폭지연(+1분)] |[도착] : 계획 01:04 -> 실제 01:04 [정시(0분)]


[지연분석]125건 발행완료

==========================================================
2026-03-10 22:48:40 서울역 기준 출발 열차 현황

==========================================================
[KTX 00117호 열차] 22:58출발 | 노선미상   |서울   ➡️ 동대구  | 곧 출발 (9분 후)- 탑승 중
[KTX 00119호 열차] 23:28출발 | 노선미상   |서울   ➡️ 대전   | 출발 39분전 (대기중)

 60초 후 갱신..

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

## 실제 겪은 트러블 슈팅

### . 🚨 [API 한계] 실시간 데이터가 빈 리스트(`[]`)로 반환되는 현상

- **문제 (Q):** 코드 로직은 완벽한데 오늘 출발하는 기차의 실시간 정보를 요청하면 계속 빈 데이터만 나옴.
- **원인:** 코레일 API의 숨겨진 제약사항. '실시간'이라는 이름과 달리, 실제 위치 데이터는 **하루가 지나야(D-1) 업데이트되는 배치(Batch)성 데이터**였음. (스크린샷과 어제 날짜 조회 테스트로 원인 규명!)
    
- **해결 (Troubleshooting):** 파이프라인 아키텍처를 전면 수정하여 투트랙(Two-track) 전략 도입.
    
    1. **전광판용 (당일):** 운행 계획(시간표)과 현재 시각을 비교하여 진행률과 상태를 계산하는 **'가짜 실시간(Simulation)' 생성기** 구축.
    2. **데이터 적재용 (익일):** 자정 이후 어제자 실제 운행 기록을 조회하여 진짜 지연 시간을 계산하는 **'지연 분석 배치(Batch)'** 로직 분리 구축.

### 2. 🚨 [API 차단] 무한 루프로 인한 트래픽 초과 (Rate Limit) 위험

- **문제 (Q):** 60초마다 도는 `while True` 루프 안에서 운행계획(Schedule)을 매번 새로 호출함.
- **원인:** 고정된 데이터를 불필요하게 반복 요청하여 공공데이터 서버 IP 차단(DDoS 의심) 위험 발생.
- **해결 (Troubleshooting):** * 운행계획은 **하루 1번만** 호출하여 클래스 메모리(`self.daily_schedule`)에 저장해 두고 재사용하도록 변경.
    - 지연 분석을 위해 수백 대의 기차를 조회할 때는 `time.sleep(0.3)`을 부여해 서버 부하 완화. 자정에 날짜가 바뀌면 스케줄을 새로 갱신하는 스마트 로직 추가.
        

### 3. 🚨 [로직 버그] "시각 정보 없음" 파싱 에러!!(한참헤맴)

- **문제 (Q):** 전광판에 상태 대신 `시각 정보 없음`이 무더기로 출력됨.
- **원인:** `datetime.strptime(f"{today}{plan_dep}", "%Y-%m-%d %H:%M")` 에서 날짜와 시간 사이에 **공백(띄어쓰기)** 이 누락되어 포맷 불일치 발생 (`ValueError`).
- **해결 (Troubleshooting):** `try-except` 문에 `print(e)`를 찍어 디버깅으로 원인을 정확히 찾아내고, `f"{today} {plan_dep}"`로 공백을 넣어 완벽히 해결.

### 4. 🚨 [로직 버그] 자정(Midnight) 통과 시 마이너스 지연 시간 발생

- **문제 (Q):** 23:50 출발 기차가 00:10에 출발했을 때 계산 오류 발생.
- **원인:** 날짜 전환을 인지하지 못한 컴퓨터가 $10 - 1430 = -1420$분 (약 23시간 일찍 출발)으로 착각함.
- **해결 (Troubleshooting):** 시간 차이(`diff`)가 `-720`분 이하이거나 `720`분 이상일 경우, **하루 전체 시간인 `1440`분을 더하거나 빼주어** 자정 크로스오버 문제를 수학적으로 완벽하게 보정함.

### 5. 🚨 [DB 충돌] Producer 데이터와 PostgreSQL 테이블 스키마 불일치

- **문제 (Q):** 기존에 짜둔 `init.sql`을 그대로 써도 되는지 의문 발생.
- **원인:** Python (Producer) 측에서 시뮬레이션을 위해 `data_type: "estimated"`, `progress_pct` 같은 새로운 키(Key)를 추가했으나, 기존 DB 테이블은 옛날 API 원본 필드만 기다리고 있었음. 또한, 지연 상태를 ENUM으로 고정해 두어 텍스트 삽입 시 충돌 예정이었음.
- **해결 (Troubleshooting):** 파이썬 코드에서 쏘는 JSON 메시지 구조와 **1:1 매칭되도록 `train_realtime`, `train_delay` 테이블 구조 전면 재설계.** (ENUM 제거, `data_type` 컬럼 추가)

### 6. 🚨 [파이썬 문법] 버전 및 문법 관련 에러 삼총사

- **문제 1 (`TypeError`):** `int | None` 문법을 사용했다가 파이썬 버전에 막힘.    
    - 👉 `from typing import Optional` 추가 후 `Optional[int]`로 우회하여 해결.
- **문제 2 (Type Hint 경고):** `_format_dt` 함수에 `None`을 넘겨주어 VS Code 경고 발생.
    - 👉 `None` 대신 빈 문자열 `""`을 넘겨주어 타입 안정성(Type Safety) 확보.

### 7. 💡 [데이터 구조 탐색] 리스트를 거꾸로(`reversed`) 찾는 이유

- **질문 (Q):** 왜 도착 시간을 찾을 때 `reversed(actual_items)`를 쓰는가?
- **답변:** 데이터가 역순(시간순)으로 쌓이기 때문! 앞쪽부터 찾으면 중간 기착지(예: 대전역)의 도착 시간을 최종 도착으로 오해할 수 있음. 
- 따라서 **배열의 맨 뒤(종착역)부터 역탐색**하여 진짜 도착 시간을 뽑아내는 데이터 엔지니어링 스킬 적용.

---

✅ 완료되면 → [[04_Spark_Streaming]] 으로 이동