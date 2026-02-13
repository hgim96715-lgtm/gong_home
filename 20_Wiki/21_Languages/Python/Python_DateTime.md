---
aliases:
  - Python 날짜
  - 시간
  - time 모듈
  - datetime
  - sleep
  - timedelta
  - isoformat
tags:
  - Python
related:
  - "[[Blocking_vs_NonBlocking]]"
  - "[[Airflow_Queues_Scaling]]"
  - "[[Python_Modules_Imports]]"
  - "[[Airflow_Sensors]]"
  - "[[Python_Mock_Data_Generator]]"
---

## 개념 한 줄 요약

데이터 엔지니어링에서 **시간(Time)** 은 두 가지 의미를 가져요.

1.  **시점 (Point in Time):** "2026년 1월 23일 14시" (로그 찍기, 스케줄링) -> `datetime` 모듈
2.  **기간/대기 (Duration/Pause):** "10초 동안 멈춰" (API 호출 간격, 폴링) -> `time` 모듈

---
## 핵심 개념 3가지 

1. **datetime (시계/달력):** "지금 몇 시 몇 분이야?" (특정 **시점**)
2. **timedelta (스톱워치/기간):** "3일 걸려", "10분 차이야" (시간의 **간격**)
3. **time.sleep (일시정지):** "잠깐 멈춰!" (프로그램 **대기**)

---
## 왜 필요한가 (Why)

왜 그냥 문자열 `"2024-01-01"`을 안 쓰고 귀찮게 이런 걸 쓸까?

**문제점:**
- 컴퓨터 입장에서 `"2024-01-01"`은 그냥 글자(String)일 뿐이야. 
- 여기서 내일 날짜를 구하려고 `"2024-01-01" + 1`을 하면 에러가 나. (글자에 숫자를 더할 순 없으니까!)

**해결책:**
- `datetime` 객체로 만들어야 "날짜 더하기/빼기" 같은 **시간 계산**이 가능해져.

---
## Practical Context (실무 활용)

**API Rate Limit 대응:**
- 외부 API가 "1초에 5번만 호출 가능" 제한이 있다면, `time.sleep(0.2)`를 넣어줘야 차단 안 당해.

**Airflow Sensor:**
- "파일이 들어왔나?" 확인하고 없으면 `time.sleep(60)` 하고 다시 확인하는 **폴링(Polling)** 구조에서 핵심이야.

- **주의점:** `time.sleep()`은 **Blocking** 함수라 너무 길게 잡으면 프로그램 전체가 멈춰버려. (이때 비동기나 `reschedule` 모드가 필요한 거지!)

>[[Blocking_vs_NonBlocking]] 문서를 보면 `sleep`이 왜 I/O 작업인지 알 수 있어.

---
## 날짜를 문자열로 바꾸기

> **"DB나 Kafka에 넣으려면 글자(String)로 바꿔야 해!"**

### ① `isoformat()`: 국제 표준 (가장 추천)

전 세계 공통 규격(**ISO 8601**)으로 바꿔줍니다.
Spark, SQL등 모든 빅데이터 시스템이 이 형식을 사랑합니다.

```python
from datetime import datetime 

now = datetime.now() 

# 사용법: 그냥 호출하면 끝! 
date_str = now.isoformat()

print(date_str) # 결과: '2024-02-13T16:30:45.123456' (중간에 T가 들어감)
```

**[장점]**

1. **문자열 정렬 = 시간 순서 정렬:** `2024-01`이 `2024-02`보다 무조건 앞에 옴. (사전순 정렬이 가능)
2. **호환성:** 별도 설정 없이 어디든 바로 들어감.


### ② strftime: 날짜 ➡ 글자 (Format,변환)

* **의미:** **Str**ing **F**ormat **Time** (시간을 예쁜 글자 포맷으로 바꿔줘)
* **용도:** 파일명 만들 때 (`data_2024-01-01.csv`), DB에 넣을 때.

> 이미 있는 **`now` (내 변수)** 한테 변신하라고 시키는 것.

```python
now = datetime.now() # (객체 상태)

# strftime: 내 변수(now)가 주어 -> (포맷 1개만 필요)
date_str = now.strftime("%Y-%m-%d") 
print(date_str) 
# 출력: "2026-01-23" (이제 더하기 빼기 불가능한 단순 글자가 됨)
```

### ③ `f-string` 포맷팅 (최신 트렌드)

`strftime`의 복잡한 `%` 기호를 외우기 귀찮을 때 씁니다.

```python
# f-string 안에서 바로 포맷 지정 가능 (: 뒤에 씀)
print(f"현재 시간: {now:%Y-%m-%d %H:%M}")
# 결과: 현재 시간: 2024-02-13 16:30
```

---
## 문자열을 날짜로 바꾸기 (Parsing)

- **의미:** **Str**ing **P**arse **Time** (글자를 분석해서 시간 객체로 만들어줘)
- **용도:** CSV 파일이나 API에서 가져온 날짜 글자를 계산 가능한 상태로 만들 때.

> 아무것도 없으니 **`datetime` (공장)** 한테 재료를 주고 만들어 달라고 해야 함. 

```python
date_text = "2026/01/23" # (글자 상태)

# "이 글자는 년(Y)/월(m)/일(d) 순서로 되어 있어"라고 알려줘야 함
# strptime: 공장(datetime)이 주어 -> (재료 2개 필요)
date_obj = datetime.strptime(date_text, "%Y/%m/%d")
print(date_obj)
# 출력: 2026-01-23 00:00:00 (이제 계산 가능!)
```

### C. 자주 쓰는 포맷 코드표 (Cheatsheet)

이거 외우지 말고 필요할 때마다 와서 보고 베껴!

|**코드**|**의미**|**예시 (2026년 1월 5일 오후 2시 30분)**|
|---|---|---|
|`%Y`|4자리 연도|`2026`|
|`%m`|2자리 월|`01` (0 채워짐)|
|`%d`|2자리 일|`05` (0 채워짐)|
|`%H`|24시간제|`14`|
|`%M`|분|`30`|
|`%S`|초|`00`|
|`%a`|요일 (짧게)|`Mon`, `Tue`|

----
## Core Components 

### datetime (날짜와 시간 그 자체)

우리가 달력을 보며 "오늘 며칠이지?" 하는 것과 같아.

```python
from datetime import datetime 

# 1. 현재 시간 가져오기
now = datetime.now() print(now) 
# 출력: 2026-01-23 14:30:00.123456 (년-월-일 시:분:초.마이크로초)
```

### timedelta (시간의 차이) 

**"Time(시간) + Delta(차이)"** 라는 뜻이야. 날짜 계산을 위한 **'시간 덩어리'** 라고 생각하면 돼.

- 숫자 `1`은 그냥 숫자지만, `timedelta(days=1)`은 **"하루(24시간)라는 시간의 양"** 을 의미해.
- **공식:** `시점(datetime) + 기간(timedelta) = 새로운 시점(datetime)`

```python
from datetime import timedelta

# "하루"라는 시간 덩어리를 만든다.
one_day = timedelta(days=1) 

# "30분"이라는 시간 덩어리를 만든다.
thirty_mins = timedelta(minutes=30)
```

### datetime 모듈 (날짜와 시각)

```python
from datetime import datetime, timedelta

# 1. 현재 시간 구하기
now = datetime.now()

# 2. 날짜 연산 (어제 날짜 구하기) -> Airflow에서 제일 많이 씀!
yesterday = now - timedelta(days=1)
print(f"어제: {yesterday.strftime('%Y-%m-%d')}")
```

### time 모듈 (실행 대기) 

**`time.sleep()`** 은 프로그램을 잠시 멈추는 기능
API 호출 제한을 피하거나 폴링할 때 씁니다. 하지만 너무 길게 잡으면 프로그램이 멈춥니다.

컴퓨터는 너무 빨라서 반복문을 돌리면 0.001초 만에 끝납니다. 
**"사람 속도"** 나 **"센서 주기"** 를 흉내 내려면 반드시 재워야 합니다.

>[[Blocking_vs_NonBlocking]] 문서를 참고

```python
import time

print("API 호출 시작...")
time.sleep(3)  # 3초 동안 아무것도 안 하고 대기 (CPU 멈춤) (Blocking I/O)
print("3초 뒤 호출 완료!")
```

###  Timezone (UTC)

서버 시간은 무조건 **UTC(협정 세계시)** 로 맞추는 것이 국룰입니다. 
한국 시간(KST)은 출력해서 보여줄 때만 변환하세요.

```python
from datetime import timezone

# UTC 현재 시간 (끝에 +00:00이 붙음)
now_utc = datetime.now(timezone.utc)
print(now_utc.isoformat())
# 결과: '2024-02-13T07:30:45.123456+00:00'
```


---
## 초보자가 자주 착각하는 포인트

1. `import datetime` vs `from datetime import datetime`:
	- 파이썬의 `datetime` **모듈(파일)** 안에 `datetime`이라는 **클래스(도구)** 가 또 들어있어서 이름이 겹쳐.
	- 그래서 보통 `from datetime import datetime`이라고 써야 `datetime.now()`처럼 바로 쓸 수 있어. 그냥 `import datetime` 하면 `datetime.datetime.now()`라고 길게 써야 해서 귀찮아.

2. **문자열이랑 헷갈리지 말기:**
	- `"2024-01-01"` (따옴표 있음) ➡ **문자열(String)**. 계산 불가. ❌
	- `datetime(2024, 1, 1)` (따옴표 없음) ➡ **객체(Object)**. 계산 가능. ⭕️