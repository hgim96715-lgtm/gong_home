---
aliases:
  - Python 날짜
  - 시간
  - time 모듈
  - datetime
  - sleep
  - timedelta
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

파이썬에서 **시간**을 다루는 건 딱 3가지 개념만 알면 돼.

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
## Core Components (핵심 부품) 이해 하기

### A. datetime (날짜와 시간 그 자체)

우리가 달력을 보며 "오늘 며칠이지?" 하는 것과 같아.

```python
from datetime import datetime 

# 1. 현재 시간 가져오기
now = datetime.now() print(now) 
# 출력: 2026-01-23 14:30:00.123456 (년-월-일 시:분:초.마이크로초)
```

### B. timedelta (시간의 차이) 

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

### C. time.sleep (잠깐 멈춤)

프로그램을 지정한 시간(초 단위)만큼 강제로 재우는 거야.

컴퓨터는 너무 빨라서 반복문을 돌리면 0.001초 만에 끝납니다. 
**"사람 속도"** 나 **"센서 주기"** 를 흉내 내려면 반드시 재워야 합니다.

```python
import time

print("라면 물 끓이는 중...")
time.sleep(3) # 3초 동안 아무것도 안 하고 대기 (CPU 멈춤)
print("라면 투입!")
```

---
## Data Transformation (날짜 ↔ 문자열 변환) 

데이터 엔지니어가 가장 많이 하는 실수: **"문자열이랑 객체를 헷갈리는 것!"**
이 두 개를 자유자재로 바꾸는 함수가 바로 `strftime`과 `strptime`이야. (철자 헷갈림 주의!)

### A. strftime: 날짜 ➡ 글자 (Format,변환)

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

### B. strptime: 글자 ➡ 날짜 (Parse, 생성)

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


---
## Code Core Points

### A. datetime 모듈 (날짜와 시각)

```python
from datetime import datetime, timedelta

# 1. 현재 시간 구하기
now = datetime.now()

# 2. 날짜 연산 (어제 날짜 구하기) -> Airflow에서 제일 많이 씀!
yesterday = now - timedelta(days=1)
print(f"어제: {yesterday.strftime('%Y-%m-%d')}")
```

### B. time 모듈 (실행 대기) 

**`time.sleep()`** 은 프로그램을 잠시 멈추는 기능이야.

```python
import time

print("API 호출 시작...")
time.sleep(3) # 3초 동안 CPU가 멈춤 (Blocking I/O)
print("3초 뒤 호출 완료!")
```

---
## 초보자가 자주 착각하는 포인트

1. `import datetime` vs `from datetime import datetime`:
	- 파이썬의 `datetime` **모듈(파일)** 안에 `datetime`이라는 **클래스(도구)** 가 또 들어있어서 이름이 겹쳐.
	- 그래서 보통 `from datetime import datetime`이라고 써야 `datetime.now()`처럼 바로 쓸 수 있어. 그냥 `import datetime` 하면 `datetime.datetime.now()`라고 길게 써야 해서 귀찮아.

2. **문자열이랑 헷갈리지 말기:**
	- `"2024-01-01"` (따옴표 있음) ➡ **문자열(String)**. 계산 불가. ❌
	- `datetime(2024, 1, 1)` (따옴표 없음) ➡ **객체(Object)**. 계산 가능. ⭕️