---
aliases:
  - Python 날짜
  - 시간
  - time 모듈
  - datetime
  - sleep
  - timedelta
  - isoformat
  - date
  - today
  - isocalendar
  - weekday
tags:
  - Python
related:
  - "[[Blocking_vs_NonBlocking]]"
  - "[[Airflow_Queues_Scaling]]"
  - "[[Python_Modules_Imports]]"
  - "[[Airflow_Sensors]]"
  - "[[Python_Mock_Data_Generator]]"
  - "[[Python_Calendar_Module]]"
  - "[[00_Python_HomePage]]"
---
# Python_DateTime

## 개념 한 줄 요약

> **시점(언제?)은 `datetime` / 기간(얼마나?)은 `timedelta` / 대기(잠깐!)는 `time.sleep`**

---

---

# 왜 문자열 말고 datetime 객체를 쓰나

```
컴퓨터 입장에서 "2024-01-01" 은 그냥 글자(String) 일 뿐

"2024-01-01" + 1  <- TypeError 발생 (글자에 숫자를 더할 수 없음)

파이썬이 이해하는 객체로 만들어야
날짜 더하기/빼기, 요일 계산 같은 시간 연산이 가능해짐
```

---

---

# 핵심 개념 4가지

|개념|비유|하는 일|
|---|---|---|
|`datetime`|시계 + 달력|"지금 몇 년 몇 월 며칠, 몇 시 몇 분이야?"|
|`date`|달력만|"오늘 며칠이야?" (시간 빼고 날짜만)|
|`timedelta`|스톱워치|"3일 걸려", "10분 차이야" (시간의 간격)|
|`time.sleep`|일시정지|"잠깐 멈춰!" (프로그램 대기)|

---

---

# ① datetime — 시계 + 달력

```python
from datetime import datetime

# 현재 날짜 + 시간
now = datetime.now()
print(now)
# 출력: 2026-01-23 14:30:00.123456

# datetime.now() 와 datetime.today() 는 거의 동일하게 사용 가능
# 차이: now()는 timezone 인자를 받을 수 있음 / today()는 항상 로컬 시간
now_local  = datetime.now()        # 로컬 시간 (동일)
now_local2 = datetime.today()      # 로컬 시간 (동일)
now_utc    = datetime.now(timezone.utc)  # UTC 시간 ← 글로벌 서비스 필수

# 특정 날짜 직접 만들기
d = datetime(2026, 12, 25, 9, 0, 0)  # 2026-12-25 09:00:00
```

```
datetime.now()   : 가장 많이 씀. timezone 인자 주면 UTC 도 가능
datetime.today() : now() 와 같지만 timezone 인자 없음
→ 둘 다 로컬 시간이면 결과 동일, now() 쓰는 습관이 낫다
```

---

---

# ② date — 날짜만

> 시간(시:분:초) 정보가 필요 없고 연-월-일만 필요할 때

```python
from datetime import date

# 오늘 날짜 (시간 정보 없음)
today = date.today()
print(today)  # 2026-02-24

# 특정 날짜 직접 만들기
d_day = date(2026, 12, 25)

# 요일 확인 (월=0, 화=1, 수=2, 목=3, 금=4, 토=5, 일=6)
print(today.weekday())  # 1 = 화요일

# 올해 몇 번째 주인지 (주간 통계에 필수)
# isocalendar() -> (연도, 주차, 요일) 반환
year, week, weekday = today.isocalendar()
print(f"오늘은 {year}년의 {week}번째 주입니다.")
```

---

---

# ③ timedelta — 시간 간격

> **공식: `시점(datetime) + 기간(timedelta) = 새로운 시점(datetime)`**

```python
from datetime import datetime, timedelta, timezone

now = datetime.now()

# 기간 만들기
one_day     = timedelta(days=1)
thirty_mins = timedelta(minutes=30)
one_hour    = timedelta(hours=1)

# 날짜 연산
yesterday  = now - timedelta(days=1)   # 어제 ← Airflow 필수 로직
tomorrow   = now + timedelta(days=1)   # 내일
last_week  = now - timedelta(weeks=1)  # 일주일 전

# UTC 시간
now_utc = datetime.now(timezone.utc)
print(now_utc.isoformat())
# 2024-02-13T07:30:45.123456+00:00
```

```
데이터 파이프라인에서 가장 많이 쓰는 패턴:
  now - timedelta(days=1)   어제 날짜 데이터 수집
  strftime('%Y%m%d')        YYYYMMDD 문자열로 변환
  → 폴더명, API 파라미터, 파티션 키로 활용
```

---

---

# ④ time.sleep — 프로그램 대기

```python
import time

print("API 호출 시작...")
time.sleep(3)  # 3초 동안 완전히 멈춤 (CPU 도 멈춤 = Blocking)
print("3초 뒤 호출 완료!")
```

```
컴퓨터는 반복문을 0.001초 만에 끝냄
"사람 속도" 나 "센서 주기" 를 흉내 내거나
API Rate Limit 초과 방지용으로 반드시 넣어야 함

주의: time.sleep() 은 CPU 를 통째로 멈추는 Blocking 함수
      비동기(async) 환경에서는 asyncio.sleep() 을 써야 함
      [[Blocking_vs_NonBlocking]] 참고
```

```python
# API Rate Limit 대응: 1초에 5번 제한 → 0.2초 간격으로 호출
for i in range(10):
    call_api()
    time.sleep(0.2)

# 폴링 구조: 파일이 들어왔나 계속 확인
while True:
    if file_exists():
        process()
        break
    time.sleep(60)  # 없으면 60초 대기 후 재확인
```

---

---

# ⑤ 날짜 → 문자열 (DB / Kafka 저장용)

> "DB 나 Kafka 에 넣으려면 String 으로 바꿔야 한다"

```python
from datetime import datetime
now = datetime.now()

# 방법 1: isoformat() — 국제 표준 (빅데이터 시스템 선호)
print(now.isoformat())
# 2026-02-24T21:06:32.123456  ← 중간에 T 가 들어감

# 방법 2: strftime() — 원하는 포맷으로
date_str = now.strftime("%Y-%m-%d")
print(date_str)           # 2026-02-24
print(type(date_str))     # <class 'str'>  ← 이제 문자열, 계산 불가

today_fmt = now.strftime("%Y%m%d")
print(today_fmt)          # 20260224  ← API 파라미터 run_ymd 형식

# 방법 3: f-string (간편)
print(f"현재 시간: {now:%Y-%m-%d %H:%M}")
# 현재 시간: 2026-02-24 21:06
```

---

---

# ⑥ 문자열 → 날짜 (계산용으로 변환)

> **`strptime` = Str**ing **P**arse **Time** — 글자를 분해해서 날짜 객체로 만든다

```python
from datetime import datetime

date_text = "2026/01/23"  # 단순 글자 상태, 계산 불가

# (글자, 포맷규칙) → 날짜 객체
date_obj = datetime.strptime(date_text, "%Y/%m/%d")

print(date_obj)         # 2026-01-23 00:00:00  ← 시간은 00시로 자동 세팅
print(date_obj.date())  # 2026-01-23  ← .date() 로 시간 꼬리 잘라내기
```

```python
# 실전: API 응답 날짜 파싱
# API 원본: "2026-04-09 05:13:00.0"
# [11:16] 하드코딩 대신 strptime 으로 안전하게 처리

raw = "2026-04-09 05:13:00.0"
try:
    dt = datetime.strptime(raw, "%Y-%m-%d %H:%M:%S.%f")
    print(dt.strftime("%H:%M"))  # 05:13
except ValueError:
    print(raw[11:16])  # 포맷 다르면 슬라이싱으로 fallback
```

> [[02_API_Producer]] `_format_dt()` 에서 실제로 사용한 패턴

---

---

# 포맷 코드 치트시트

> 외우지 말고 필요할 때 보고 베껴

|코드|의미|예시 (2026-01-05 14:30:00)|
|---|---|---|
|`%Y`|4자리 연도|`2026`|
|`%m`|2자리 월 (0 채움)|`01`|
|`%d`|2자리 일 (0 채움)|`05`|
|`%H`|24시간제|`14`|
|`%M`|분|`30`|
|`%S`|초|`00`|
|`%f`|마이크로초|`000000`|
|`%a`|요일 (짧게)|`Mon`, `Tue`|
|`%Y%m%d`|API 파라미터용|`20260105`|
|`%Y-%m-%d`|DB 저장용|`2026-01-05`|
|`%H:%M`|시:분|`14:30`|

---

---

# 실무 활용 패턴

```python
from datetime import datetime, timedelta, timezone

# Airflow 스케줄링 — 어제 날짜 폴더명 만들기
yesterday = datetime.now() - timedelta(days=1)
partition = yesterday.strftime("ds=%Y-%m-%d")  # ds=2026-02-23

# API 파라미터 run_ymd 만들기
today_str = datetime.now().strftime("%Y%m%d")  # 20260224

# UTC 저장, KST 는 화면에만
now_utc = datetime.now(timezone.utc)
now_kst = now_utc + timedelta(hours=9)
print(f"저장: {now_utc.isoformat()}")   # UTC 로 DB 저장
print(f"표시: {now_kst:%Y-%m-%d %H:%M} KST")  # 화면 출력용
```

---

---

# 초보자 착각 포인트

```
① import datetime vs from datetime import datetime

  import datetime         → datetime.datetime.now()  길게 써야 함
  from datetime import datetime → datetime.now()      바로 사용 가능
  → 보통 from datetime import datetime, timedelta, timezone 으로 한 번에 가져옴

② 문자열이랑 헷갈리지 말기

  "2024-01-01"          (따옴표 있음) → String. 계산 불가 ❌
  datetime(2024, 1, 1)  (따옴표 없음) → 객체. 계산 가능 ⭕️

③ datetime.now() vs datetime.today()

  둘 다 로컬 시간 → 결과 동일
  차이: now() 는 timezone 인자를 받을 수 있음
  datetime.now(timezone.utc)  → UTC 시간
  datetime.today()            → timezone 인자 없음 (항상 로컬)
  → now() 쓰는 습관 들이는 게 낫다
```